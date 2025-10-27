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

## Simplicity Profiles

**This section shows how the same universal Parser interface is implemented differently based on usage scale.**

### Profile 1: Personal Use (Finance App - Single Bank, Hardcoded Parser)

**Context:** Darwin only uploads Bank of America PDFs (single source_type), hardcoded BoFA parser

**Configuration:**
```yaml
parser:
  parser_id: "bofa_pdf_parser"  # Hardcoded (only parser in app)
  version: "1.0.0"  # No versioning (single implementation)
  supported_source_types: ["bofa_pdf"]  # Only BoFA supported
  settings:
    strict_mode: false  # Warnings OK (don't block parsing)
    max_rows: 1000  # Reasonable limit for personal statements
```

**What's Used:**
- ✅ Core `parse()` method - Extract transactions from BoFA PDF
- ✅ AS-IS extraction - No validation (date/amount as strings)
- ✅ Zero rows = success - Empty statement (no transactions) is valid
- ✅ Basic error handling - `ParseError` if PDF corrupted
- ✅ Deterministic row IDs - Same file → same row_id assignments

**What's NOT Used:**
- ❌ Parser registry - Only 1 parser (BoFA), hardcoded
- ❌ Versioning system - No parser version tracking
- ❌ `can_parse()` detection - Only uploads BoFA PDFs (no auto-detection needed)
- ❌ Warnings vs errors distinction - All issues are errors (simpler)
- ❌ Partial parse support - Either parse all rows or fail
- ❌ Parser performance metrics - No latency tracking

**Implementation Complexity:** **LOW**
- ~200 lines Python
- Single file: `bofa_pdf_parser.py`
- Uses PyPDF2 for text extraction
- Regex patterns for transaction rows
- No tests (manual verification with user's own PDFs)

**Narrative Example:**
> Darwin uploads "BoFA_Oct2024.pdf". API calls `BoFAPDFParser().parse(pdf_bytes)`. Parser:
> 1. Extract text from PDF using PyPDF2
> 2. Find transaction table (regex search for "Date Description Amount Balance")
> 3. Extract rows with pattern: `r'(\d{2}/\d{2}/\d{4})\s+(.+?)\s+(-?\$[\d,]+\.\d{2})\s+(\$[\d,]+\.\d{2})'`
> 4. Create Observation for each row: `{"raw_data": {"date": "10/15/2024", "description": "WHOLE FOODS MARKET", "amount": "-$87.43", "balance": "$4,462.89"}, "row_id": 0}`
> 5. Return 42 observations (42 transactions in October)
>
> If PDF corrupted → PyPDF2 raises exception → Wrapped as `ParseError("Failed to extract text from PDF")` → Upload marked as `error`.

---

### Profile 2: Small Business (Accounting Firm - Multiple Parsers, Basic Versioning)

**Context:** Accountants upload docs from 10 different banks + client invoices (multiple source_types), need parser selection

**Configuration:**
```yaml
parser_registry:
  parsers:
    - parser_id: "bofa_pdf_parser"
      version: "1.2.0"
      supported_source_types: ["bofa_pdf"]
    - parser_id: "chase_pdf_parser"
      version: "1.0.0"
      supported_source_types: ["chase_pdf"]
    - parser_id: "wells_fargo_csv_parser"
      version: "1.1.0"
      supported_source_types: ["wells_fargo_csv"]
    - parser_id: "generic_invoice_parser"
      version: "2.0.0"
      supported_source_types: ["invoice_pdf", "invoice_docx"]
  settings:
    strict_mode: false
    max_rows: 5000
    warnings_as_errors: false
    partial_parse_allowed: true  # Return rows before corruption
```

**What's Used:**
- ✅ Parser registry - Lookup parser by `source_type`
- ✅ Multiple parsers - 4 different parsers (BoFA, Chase, Wells Fargo, invoices)
- ✅ Basic versioning - Track parser version in parse log
- ✅ `can_parse()` detection - Auto-detect if wrong parser selected
- ✅ Warnings - Non-fatal issues logged but parsing continues
- ✅ Partial parse support - If row 45 corrupted, return rows 0-44
- ✅ Zero rows = success

**What's NOT Used:**
- ❌ A/B testing - Single parser per source_type (no canary deployments)
- ❌ Parser metrics - Just basic logging
- ❌ Schema evolution tracking - No automatic migration

**Implementation Complexity:** **MEDIUM**
- ~800 lines Python (4 parsers × ~200 lines each)
- Simple registry: `dict[source_type -> Parser]`
- Basic versioning: Log parser version in ParseLog
- Unit tests per parser (golden files for regression testing)

**Narrative Example:**
> Accountant uploads "chase_statement.pdf" with `source_type="chase_pdf"`. API looks up parser: `ParserRegistry.get_parser("chase_pdf")` → Returns `ChasePDFParser(version="1.0.0")`. Calls `parser.parse(pdf_bytes)`:
> 1. Extract 45 transactions (rows 0-44) successfully
> 2. Row 45 has corrupted text (mangled OCR) → Warning: "Row 45: unparseable, skipping"
> 3. Continue parsing rows 46-50 (5 more transactions)
> 4. Return `ParseResult(success=True, observations=[...50 total], warnings=["Row 45: unparseable, skipping"])`
> 5. ParseLog records: `{"parser_id": "chase_pdf_parser", "version": "1.0.0", "rows_extracted": 50, "warnings": 1}`
>
> Accountant sees: "Parsed 50 transactions, 1 warning (row 45 skipped)". Can review warning later, but upload succeeds.

---

### Profile 3: Enterprise (Bank - Parser Versioning, Canary Deployments, Metrics)

**Context:** Bank processes 10,000 uploads/day from 50+ bank/credit card formats, needs safe parser updates (canary deployments), A/B testing

**Configuration:**
```yaml
parser_registry:
  parsers:
    - parser_id: "universal_credit_card_parser"
      versions:
        - version: "2.1.0"
          rollout_percentage: 5  # Canary: 5% of traffic
          enabled: true
        - version: "2.0.0"
          rollout_percentage: 95  # Stable: 95% of traffic
          enabled: true
        - version: "1.9.0"
          enabled: false  # Deprecated
      supported_source_types: ["visa_pdf", "mastercard_pdf", "amex_pdf"]
  settings:
    strict_mode: false
    max_rows: 100000  # Credit card statements can be huge
    warnings_as_errors: false
    partial_parse_allowed: true
    canary_monitoring:
      error_rate_threshold: 0.05  # Rollback if >5% error rate
      comparison_window_hours: 24
    metrics:
      enabled: true
      backend: "prometheus"
      labels: ["parser_id", "version", "source_type"]
    schema_evolution:
      track_field_changes: true  # Detect new fields in output
      alert_on_breaking_changes: true
```

**What's Used:**
- ✅ Parser registry with versioning
- ✅ Canary deployments - 5% traffic to v2.1.0 (new), 95% to v2.0.0 (stable)
- ✅ A/B testing - Compare error rates v2.1.0 vs v2.0.0
- ✅ Automatic rollback - If v2.1.0 error rate >5%, route all traffic to v2.0.0
- ✅ `can_parse()` detection - Verify parser recognizes format
- ✅ Warnings - Extensive warning categories
- ✅ Partial parse - Critical for large statements (100K+ rows)
- ✅ Prometheus metrics - Track latency, error rate, warnings per parser version
- ✅ Schema evolution tracking - Alert if parser outputs new fields

**What's NOT Used:**
- (All features enabled at enterprise scale)

**Implementation Complexity:** **HIGH**
- ~5000 lines TypeScript/Python
- Universal parser (handles 50+ formats with heuristics)
- Canary deployment logic (route 5% → v2.1.0, monitor error rates)
- Comprehensive test suite (1000+ golden files covering edge cases)
- Schema evolution detector (compare field sets across versions)
- Monitoring dashboards: Error rate by parser version, latency p95, warning categories

**Narrative Example:**
> Customer uploads Visa statement PDF. API routes to ParserRegistry:
> 1. Lookup parser: `source_type="visa_pdf"` → `UniversalCreditCardParser`
> 2. **Canary routing**: Random selection → 5% chance → Selects v2.1.0 (canary), 95% chance → Selects v2.0.0 (stable). Random draw: 0.03 → **v2.1.0 selected**
> 3. Call `parser_v2_1_0.parse(pdf_bytes)`:
>    - Extract 1,200 transactions
>    - Row 450: Date format unusual ("Oct 15, 2024" instead of "10/15/2024") → Warning: "Date format variant detected"
>    - Row 800: Balance missing (field blank) → Warning: "Balance field empty"
>    - Return `ParseResult(success=True, observations=1200, warnings=["Date format variant: row 450", "Balance missing: row 800"])`
> 4. Prometheus metrics recorded:
>    - `parser_duration_seconds{parser_id="universal_credit_card",version="2.1.0",source_type="visa_pdf"} 3.2`
>    - `parser_observations_total{parser_id="universal_credit_card",version="2.1.0"} 1200`
>    - `parser_warnings_total{parser_id="universal_credit_card",version="2.1.0"} 2`
>    - `parser_errors_total{parser_id="universal_credit_card",version="2.1.0"} 0`
> 5. ParseLog: `{"parser_id":"universal_credit_card","version":"2.1.0","upload_id":"UL_12345","rows_extracted":1200,"warnings":2,"latency_ms":3200}`
>
> **Canary monitoring (runs every hour)**:
> - Query Prometheus: Compare error rates v2.1.0 vs v2.0.0 (last 24 hours)
> - v2.1.0 error rate: 2.1% (21 failures / 1000 uploads)
> - v2.0.0 error rate: 1.8% (171 failures / 9500 uploads)
> - Delta: +0.3% (within threshold of 5%) ✅ → **Canary healthy, continue rollout**
>
> Alternative scenario: v2.1.0 has bug (regex broken for certain date formats) → Error rate spikes to 12% → Monitoring detects `error_rate > 5%` threshold → **Automatic rollback**: Update registry: `{version: "2.1.0", rollout_percentage: 0, enabled: false}`, `{version: "2.0.0", rollout_percentage: 100}` → Alert: "P1: Parser v2.1.0 rolled back due to high error rate (12%)" → Engineering investigates.

---

**Key Insight:** The same `Parser` interface works across all 3 profiles. Personal use has 1 hardcoded parser (BoFA only); Small business has parser registry with multiple parsers; Enterprise adds canary deployments, A/B testing, and automatic rollback based on error rate monitoring.

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

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Extract raw transaction rows from bank statement PDFs
**Example:** Bank of America PDF statement (42 rows) → `bofa_pdf_parser` extracts observations with raw_data `{"date": "01/15/2024", "amount": "-5.75", "description": "  STARBUCKS #1234  "}` (AS-IS strings, no validation) → 42 ObservationTransaction records in ObservationStore
**Parser types:** `bofa_pdf_parser`, `chase_csv_parser`, `amex_ofx_parser`, `venmo_json_parser`
**Observations extracted:** `ObservationTransaction` (date, amount, description, currency, account)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Extract lab test results from clinical lab report PDFs
**Example:** Quest Diagnostics lab report PDF (8 tests) → `quest_pdf_parser` extracts observations with raw_data `{"test": "Glucose", "value": "95", "unit": "mg/dL", "date": "2024-03-15"}` (AS-IS strings) → 8 ObservationLabResult records
**Parser types:** `quest_pdf_parser`, `labcorp_pdf_parser`, `hl7_message_parser`
**Observations extracted:** `ObservationLabResult` (test_name, value, unit, normal_range, test_date)
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ Legal
**Use case:** Extract clauses from contract PDFs
**Example:** Contract PDF (25 pages, 47 clauses) → `contract_pdf_parser` extracts observations with raw_data `{"clause_num": "3.2", "text": "Payment due within 30 days...", "page": 5}` (AS-IS text) → 47 ObservationClause records
**Parser types:** `contract_pdf_parser`, `court_filing_pdf_parser`, `deposition_transcript_parser`
**Observations extracted:** `ObservationClause` (clause_number, text, page_number, section)
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ RSRCH (Utilitario Research)
**Use case:** Extract founder investment facts from web articles (TechCrunch, Bloomberg)
**Example:** TechCrunch article HTML (3,500 words) → `web_fact_parser` extracts observations with raw_data `{"text": "@sama invested $375M in OpenAI", "entity": "@sama", "amount": "$375M", "company": "OpenAI", "date": "2024-02-25"}` (AS-IS text extraction) → 12 RawFact records (article mentions multiple founders/investments)
**Parser types:** `web_fact_parser` (TechCrunch, Bloomberg, Medium), `tweet_json_parser` (Twitter API), `podcast_transcript_parser` (Lex Fridman transcripts)
**Observations extracted:** `RawFact` (subject_entity, claim, fact_type, source_url, publication_date)
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ E-commerce
**Use case:** Extract product listings from supplier catalog CSVs
**Example:** Supplier catalog CSV (1,500 products) → `supplier_csv_parser` extracts observations with raw_data `{"sku": "IPHONE15-256-BLU", "title": "  iPhone 15 Pro Max - 256GB  ", "price": "$1,199.99", "category": "Electronics > Phones"}` (AS-IS strings, with spaces) → 1,500 ObservationProduct records
**Parser types:** `supplier_csv_parser`, `amazon_api_parser`, `shopify_export_parser`
**Observations extracted:** `ObservationProduct` (SKU, title, price, category, description, image_url)
**Status:** ✅ Conceptually validated via examples in this doc

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (universal Parser interface with parse() method, domain-specific implementations)
**Reusability:** High (same Parser interface works for PDF, CSV, JSON, XML, HTML; only parse() logic differs per source format)

---

## Related Primitives

- **ParserRegistry**: Discovers and selects appropriate parser for `source_type`
- **ObservationStore**: Stores raw observations extracted by parser
- **ParseLog**: Records parser execution details
- **Normalizer** (Vertical 1.3): Validates and transforms parser output

---

**Last Updated**: 2025-10-27
**Maturity**: Spec complete, ready for implementation
