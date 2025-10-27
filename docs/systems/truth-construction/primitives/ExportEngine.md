# ExportEngine (OL Primitive)

## Definition

**ExportEngine** generates downloadable files (CSV, PDF) from query results. It provides a standardized export interface for any canonical data, with format-specific rendering and streaming support for large datasets.

**Problem it solves:**
- Manual export logic scattered across codebase (inconsistent formats)
- Memory overflow on large exports (loading all rows at once)
- Missing context in exports (no filter criteria, no metadata)
- Poor PDF formatting (raw data dumps, not human-readable)

**Solution:**
- Streaming export (process rows incrementally, not all-in-memory)
- Format-specific renderers (CSV, PDF, Excel, JSON)
- Metadata inclusion (filters applied, export date, row count)
- Templates for professional PDFs (headers, footers, page numbers)

---

## Interface Contract

```python
from typing import Iterator, Dict, Any, List
from dataclasses import dataclass
from datetime import datetime

@dataclass
class ExportConfig:
    """Export configuration"""
    format: str                     # "csv" | "pdf" | "xlsx" | "json"
    filename: str                   # e.g., "transactions-2025-04.csv"
    filters_applied: Dict[str, Any] # For metadata section
    columns: List[str]              # Which columns to include
    include_summary: bool = True    # Show summary row (totals)
    include_metadata: bool = True   # Show filter criteria, export date

@dataclass
class ExportResult:
    """Export result"""
    file_path: str         # Temporary file path
    content_type: str      # MIME type (e.g., "text/csv")
    size_bytes: int        # File size
    row_count: int         # Number of rows exported

class ExportEngine:
    """
    Universal export engine for canonical data.

    Domain-agnostic - works with any tabular data.
    """

    def export_csv(
        self,
        rows: Iterator[Dict[str, Any]],
        config: ExportConfig
    ) -> ExportResult:
        """
        Exports data to CSV format (streaming)

        Args:
            rows: Iterator of row dicts (for memory efficiency)
            config: Export configuration

        Returns:
            ExportResult with file path and metadata

        Example:
            rows = db.stream_query("SELECT * FROM canonical_transactions ...")
            result = engine.export_csv(
                rows=rows,
                config=ExportConfig(
                    format="csv",
                    filename="transactions-2025-04.csv",
                    filters_applied={"date_from": "2025-04-01"},
                    columns=["date", "merchant", "amount", "category"]
                )
            )
            # → CSV file at result.file_path
        """

    def export_pdf(
        self,
        rows: Iterator[Dict[str, Any]],
        config: ExportConfig
    ) -> ExportResult:
        """
        Exports data to PDF format with styling

        Features:
        - Header with export metadata (filters, date, user)
        - Table with alternating row colors
        - Summary row with totals
        - Page numbers
        - Footer with generation timestamp

        Example:
            result = engine.export_pdf(
                rows=rows,
                config=ExportConfig(
                    format="pdf",
                    filename="transactions-2025-04.pdf",
                    filters_applied={"date_from": "2025-04-01", "category": "Business"},
                    columns=["date", "merchant", "amount"],
                    include_summary=True
                )
            )
        """

    def export_excel(
        self,
        rows: Iterator[Dict[str, Any]],
        config: ExportConfig
    ) -> ExportResult:
        """
        Exports data to Excel format (.xlsx)

        Features:
        - Multiple sheets (data + metadata + summary)
        - Formatted numbers (currency, dates)
        - Freeze panes (header row)
        - Auto-column width
        """

    def stream_export(
        self,
        query_fn: callable,
        config: ExportConfig,
        chunk_size: int = 1000
    ) -> ExportResult:
        """
        Streams large exports without loading all data in memory

        Args:
            query_fn: Function that returns iterator of rows
                     (e.g., lambda: db.stream_query(...))
            config: Export configuration
            chunk_size: Rows to process at once (default: 1000)

        Example:
            # Export 100k rows without OOM
            result = engine.stream_export(
                query_fn=lambda: db.stream_query("SELECT * ..."),
                config=ExportConfig(format="csv", ...)
            )
        """
```

---

## Multi-Domain Applicability

**Universal Pattern:** Export structured data to downloadable files

### Finance Domain
```python
engine = ExportEngine()

# Export transactions for CPA
result = engine.export_csv(
    rows=canonical_store.query(
        filters={"date_from": "2024-01-01", "date_to": "2024-12-31", "deductible": True}
    ),
    config=ExportConfig(
        format="csv",
        filename="deductible-expenses-2024.csv",
        filters_applied={"year": 2024, "deductible": True},
        columns=["date", "merchant", "amount", "category", "account"],
        include_summary=True  # Total deductible amount
    )
)
# → CSV with $52,348 total deductible expenses
```

### Healthcare Domain
```python
# Export lab results for doctor
result = engine.export_pdf(
    rows=lab_store.query(filters={"patient_id": "P12345", "is_normal": False}),
    config=ExportConfig(
        format="pdf",
        filename="patient-abnormal-results.pdf",
        filters_applied={"patient": "John Doe", "abnormal_only": True},
        columns=["test_date", "test_name", "value", "reference_range", "status"]
    )
)
# → Professional PDF for medical review
```

### Legal Domain
```python
# Export high-risk contract clauses
result = engine.export_excel(
    rows=clause_store.query(filters={"risk_level": "high"}),
    config=ExportConfig(
        format="xlsx",
        filename="high-risk-clauses.xlsx",
        filters_applied={"risk_level": "high"},
        columns=["contract_id", "clause_text", "risk_reason", "counterparty"]
    )
)
# → Excel with formatting for legal team
```

### Research Domain (RSRCH - Utilitario)
```python
# Export founder facts
result = engine.export_csv(
    rows=fact_store.query(filters={"subject_entity": "Sam Altman", "fact_type": "investment"}),
    config=ExportConfig(
        format="csv",
        filename="sam-altman-investments-2020-2025.csv",
        filters_applied={"subject_entity": "Sam Altman", "fact_type": "investment"},
        columns=["claim", "subject_entity", "investment_amount", "discovered_at", "source_url", "source_credibility"]
    )
)
# → Fact list for VC firm investment analysis
```

### Manufacturing Domain
```python
# Export QC failures for audit
result = engine.export_pdf(
    rows=qc_store.query(filters={"passed_qc": False, "product": "SKU-123"}),
    config=ExportConfig(
        format="pdf",
        filename="qc-failures-sku123.pdf",
        filters_applied={"product": "SKU-123", "failed_only": True},
        columns=["date", "batch_number", "measurement", "expected", "actual", "delta"]
    )
)
# → QC failure report for regulatory compliance
```

### Media Domain
```python
# Export transcript for editor
result = engine.export_csv(
    rows=snippet_store.query(filters={"speaker": "Guest-A", "sentiment": "positive"}),
    config=ExportConfig(
        format="csv",
        filename="guest-a-positive-quotes.csv",
        filters_applied={"speaker": "Guest A", "sentiment": "positive"},
        columns=["timestamp", "text", "sentiment_score", "topic"]
    )
)
# → Quotes for highlight reel
```

---

## Responsibilities

✅ **DOES:**
- Generate CSV files (streaming for large datasets)
- Generate PDF files (with styling, headers, footers)
- Generate Excel files (formatted, multi-sheet)
- Include export metadata (filters, date, row count)
- Calculate summary rows (totals, averages)
- Stream large exports (no memory overflow)
- Format data (currency, dates, numbers)
- Handle multi-currency (show original + converted)

---

## NOT Responsibilities

❌ **DOES NOT:**
- Execute queries (receives Iterator[Row] from caller)
- Store export files permanently (returns temp file path)
- Email exports (caller handles delivery)
- Compress files (caller can zip if needed)
- Apply filters (receives pre-filtered rows)
- Transform data (exports AS-IS from rows)

---

## Implementation Notes

### CSV Format

**Structure:**
```csv
# Metadata section (if include_metadata=True)
Export Date,2025-05-23 14:30:00 UTC
Filters,"date_from: 2025-04-01, category: Business > Software"
Total Rows,15

# Data section
date,merchant,amount,currency,account,category
2025-05-05,OpenAI ChatGPT,200.00,USD,bofa_credit,Business > Software
2025-05-01,Perplexity.AI,20.00,USD,bofa_credit,Business > Software
...

# Summary section (if include_summary=True)
TOTAL,,540.00,USD,,
```

**Implementation:**
```python
import csv
from io import StringIO

def export_csv(rows, config):
    output = StringIO()
    writer = csv.DictWriter(output, fieldnames=config.columns)

    # Metadata
    if config.include_metadata:
        writer.writerow({"date": "Export Date", "merchant": datetime.now().isoformat()})
        # ... more metadata

    # Header
    writer.writeheader()

    # Rows (streaming)
    total_amount = 0
    row_count = 0
    for row in rows:
        writer.writerow({col: row[col] for col in config.columns})
        total_amount += row.get("amount", 0)
        row_count += 1

    # Summary
    if config.include_summary:
        writer.writerow({"date": "TOTAL", "amount": total_amount})

    return output.getvalue()
```

---

### PDF Format

**Libraries:**
- ReportLab (Python)
- wkhtmltopdf (HTML → PDF)
- PDFKit

**Template:**
```
┌────────────────────────────────────────┐
│ Transaction Export                     │
│ Date: 2025-05-23                       │
│ Filters: Apr 2025, Business > Software│
├────────────────────────────────────────┤
│ Date       Merchant      Amount  Cat   │
├────────────────────────────────────────┤
│ 05/05/25   OpenAI        $200    Soft  │
│ 05/01/25   Perplexity    $20     Soft  │
│ ...                                    │
├────────────────────────────────────────┤
│ TOTAL                    $540          │
└────────────────────────────────────────┘
Page 1 of 1                 Generated: ...
```

**Styling:**
- Alternating row colors (zebra striping)
- Bold headers
- Currency formatting
- Page breaks (if > 50 rows/page)

---

### Streaming for Large Exports

**Problem:** Exporting 100k rows loads all in memory (OOM)

**Solution:** Process in chunks

```python
def stream_export(query_fn, config, chunk_size=1000):
    with open(f"/tmp/{config.filename}", "w") as f:
        writer = csv.writer(f)

        # Header
        writer.writerow(config.columns)

        # Stream rows
        row_count = 0
        for chunk in query_fn(chunk_size=chunk_size):
            for row in chunk:
                writer.writerow([row[col] for col in config.columns])
                row_count += 1

        return ExportResult(
            file_path=f.name,
            content_type="text/csv",
            size_bytes=f.tell(),
            row_count=row_count
        )
```

---

### Multi-Currency Handling

**Example:**
```csv
date,merchant,amount,currency,amount_usd
2025-05-05,OpenAI,200.00,USD,200.00
2025-04-26,Aeromexico,4516.00,MXN,231.78
```

**Summary:**
```csv
TOTAL,,4716.00 (mixed),, 431.78 USD
```

---

## Related Primitives

**Uses:**
- **TransactionQuery** - Gets rows to export (but doesn't query itself)
- **CanonicalStore** - Source of data (via query results)

**Used by:**
- API endpoints (GET /api/transactions/export)
- Scheduled reports (nightly email exports)
- Backup systems (export all data for archival)

---

## Version History

- **v1.0** (2025-05-23): Initial release
  - CSV export
  - PDF export (basic)
  - Streaming support
  - Metadata inclusion
