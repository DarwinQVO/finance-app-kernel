# SnapshotExporter (OL Primitive)

## Definition

**SnapshotExporter** generates PDF snapshots of dashboard state (metrics + configuration + optional transaction table). It creates reproducible reports that capture the exact state of a dashboard view at a point in time.

**Problem it solves:**
- Users need to share dashboard state with accountants/stakeholders
- No way to preserve exact dashboard configuration for audit trail
- Manual PDF generation is inconsistent and missing key metadata
- Export needs to be reproducible (same config → same PDF)

**Solution:**
- Generate PDF with dashboard metrics + configuration
- Include view metadata (date range, filters applied)
- Optionally include full transaction table
- Add signature line for accountant handoff
- Reproducible (same input → same output)

---

## Interface Contract

```python
from typing import Optional
from dataclasses import dataclass

@dataclass
class SnapshotExportOptions:
    """Options for PDF export"""
    include_transactions: bool = True  # Include transaction table
    include_signature_line: bool = False  # Add signature line for CPA
    max_transactions: int = 10000  # Limit transaction table size
    format: str = "pdf"  # "pdf" or "csv" (future)
    logo_url: Optional[str] = None  # Company logo for PDF header

@dataclass
class SnapshotExportResult:
    """Result of export operation"""
    file_bytes: bytes  # PDF binary content
    filename: str  # Suggested filename
    file_size_bytes: int
    generation_time_ms: int

class SnapshotExporter:
    """
    Generate PDF snapshots of dashboard state.

    Domain-agnostic - works for ANY dashboard configuration.
    """

    def export_snapshot(
        self,
        view_id: str,
        metrics: dict,
        user_id: str,
        options: SnapshotExportOptions = SnapshotExportOptions()
    ) -> SnapshotExportResult:
        """
        Generate PDF snapshot of dashboard.

        Args:
            view_id: Which view to export
            metrics: Pre-calculated metrics (from DashboardEngine)
            user_id: Current user (for access control + metadata)
            options: Export options

        Returns:
            SnapshotExportResult with PDF bytes

        Raises:
            ExportTooLargeError: If transaction count > max_transactions
            ExportGenerationError: If PDF generation fails
        """

    def generate_pdf(
        self,
        view_name: str,
        view_config: dict,
        metrics: dict,
        transactions: Optional[List[dict]] = None,
        metadata: Optional[dict] = None
    ) -> bytes:
        """
        Low-level PDF generation.

        PDF Structure:
        1. Header (view name, date range, generated timestamp)
        2. Summary metrics (income, expenses, net, etc.)
        3. Metric breakdowns (category, merchants, etc.)
        4. Transaction table (if include_transactions=True)
        5. Footer (filters applied, signature line if requested)
        """

    def generate_csv(
        self,
        transactions: List[dict],
        filename: str
    ) -> bytes:
        """
        Generate CSV export (alternative to PDF).

        CSV format matches 2.1 TransactionTable export.
        """
```

---

## Multi-Domain Applicability

### Finance Domain
```python
# Tax report for accountant
result = exporter.export_snapshot(
    view_id="view_q1_deductibles",
    metrics={
        "deductible_summary": {"total": 12340.00, "percentage": 42},
        "expense_by_category": {"total": 29400.00, "breakdown": [...]}
    },
    user_id="usr_eugenio",
    options=SnapshotExportOptions(
        include_transactions=True,
        include_signature_line=True  # For CPA sign-off
    )
)
# Generates: "Q1-2025-Deductibles.pdf" with signature line
```

### Healthcare Domain
```python
# Patient summary for doctor
result = exporter.export_snapshot(
    view_id="view_patient_vitals",
    metrics={
        "avg_glucose": 120,
        "avg_blood_pressure": "125/80",
        "abnormal_tests": 3
    },
    user_id="doc_smith",
    options=SnapshotExportOptions(
        include_transactions=True,  # Lab result table
        include_signature_line=True  # For doctor signature
    )
)
# Generates: "Patient-Summary-May-2025.pdf"
```

### Legal Domain
```python
# Case summary for client
result = exporter.export_snapshot(
    view_id="view_case_summary",
    metrics={
        "cases_won": 45,
        "total_hours": 320,
        "hours_by_client": [...]
    },
    user_id="atty_jones",
    options=SnapshotExportOptions(
        include_transactions=True,  # Case table
        include_signature_line=False
    )
)
# Generates: "Case-Summary-Q1-2025.pdf"
```

### Research Domain
```python
# Publication record for grant application
result = exporter.export_snapshot(
    view_id="view_publications_2025",
    metrics={
        "papers_published": 12,
        "citation_count": 450,
        "h_index": 18
    },
    user_id="researcher_123",
    options=SnapshotExportOptions(
        include_transactions=True,  # Paper table
        include_signature_line=False
    )
)
# Generates: "Publication-Record-2025.pdf"
```

### Manufacturing Domain
```python
# QC compliance report
result = exporter.export_snapshot(
    view_id="view_qc_monthly",
    metrics={
        "units_produced": 10000,
        "defect_rate": 0.0087,
        "pass_fail_ratio": "99.13%"
    },
    user_id="qc_manager",
    options=SnapshotExportOptions(
        include_transactions=True,  # QC measurement table
        include_signature_line=True  # For auditor sign-off
    )
)
# Generates: "QC-Report-May-2025.pdf"
```

### Media Domain
```python
# Analytics report for stakeholders
result = exporter.export_snapshot(
    view_id="view_q1_analytics",
    metrics={
        "total_views": 450000,
        "avg_watch_time": 180,
        "engagement_rate": 0.15
    },
    user_id="creator_123",
    options=SnapshotExportOptions(
        include_transactions=True,  # Top content table
        include_signature_line=False
    )
)
# Generates: "Q1-2025-Analytics.pdf"
```

---

## Responsibilities

**SnapshotExporter is responsible for:**
- ✅ Generating PDF from dashboard metrics + config
- ✅ Including metadata (date range, filters, generated timestamp)
- ✅ Formatting transaction table (if requested)
- ✅ Adding signature line (if requested)
- ✅ Handling multi-currency display (show both currencies)
- ✅ Generating suggested filename

---

## NOT Responsibilities

**SnapshotExporter is NOT responsible for:**
- ❌ Calculating metrics (DashboardEngine's job)
- ❌ Querying transactions (TransactionQuery's job)
- ❌ Storing exports (user downloads immediately)
- ❌ Sending exports via email (defer to v2)
- ❌ Rendering dashboard UI (DashboardGrid's job)

---

## Implementation Notes

**PDF Generation (using ReportLab):**
```python
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Table, Paragraph
from reportlab.lib.styles import getSampleStyleSheet

def generate_pdf(self, view_name, view_config, metrics, transactions=None, metadata=None):
    buffer = BytesIO()
    doc = SimpleDocTemplate(buffer, pagesize=letter)
    story = []
    styles = getSampleStyleSheet()

    # 1. Header
    story.append(Paragraph(f"<b>{view_name}</b>", styles['Title']))
    story.append(Paragraph(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}", styles['Normal']))
    story.append(Paragraph(f"Date Range: {view_config['date_range']}", styles['Normal']))

    # 2. Summary Metrics
    story.append(Paragraph("<b>Summary</b>", styles['Heading2']))
    summary_data = [
        ["Income", f"${metrics['income_summary']['total']:,.2f}"],
        ["Expenses", f"${metrics['expense_summary']['total']:,.2f}"],
        ["Net", f"${metrics['net_summary']['net']:,.2f}"]
    ]
    story.append(Table(summary_data))

    # 3. Metric Breakdowns
    story.append(Paragraph("<b>Expense by Category</b>", styles['Heading2']))
    category_data = [["Category", "Amount", "Count"]]
    for cat in metrics['expense_by_category']['breakdown']:
        category_data.append([cat['category'], f"${cat['amount']:,.2f}", str(cat['count'])])
    story.append(Table(category_data))

    # 4. Transaction Table (if requested)
    if transactions:
        story.append(Paragraph("<b>Transactions</b>", styles['Heading2']))
        txn_data = [["Date", "Merchant", "Amount", "Category"]]
        for txn in transactions[:10000]:  # Limit to 10k
            txn_data.append([
                txn['transaction_date'],
                txn['merchant'],
                f"${txn['amount']:,.2f}",
                txn['category']
            ])
        story.append(Table(txn_data))

    # 5. Footer
    if metadata and metadata.get('include_signature_line'):
        story.append(Paragraph("<br/><br/>", styles['Normal']))
        story.append(Paragraph("Signature: _______________________  Date: _______", styles['Normal']))

    doc.build(story)
    return buffer.getvalue()
```

**Filename Generation:**
```python
def generate_filename(view_name: str, date_range: dict) -> str:
    """
    Generate suggested filename from view name and date range.

    Examples:
    - "Q1 2025 Deductibles" + 2025-01-01 to 2025-03-31 → "Q1-2025-Deductibles.pdf"
    - "Monthly Overview" + current_month → "Monthly-Overview-May-2025.pdf"
    """
    slug = view_name.replace(" ", "-")
    if isinstance(date_range, dict):
        date_str = f"{date_range['from']}-to-{date_range['to']}"
    else:
        date_str = date_range  # "current_month", etc.

    return f"{slug}-{date_str}.pdf"
```

**Performance:**
- PDF generation: ~1-2s for metrics-only
- PDF with 1k transactions: ~2-3s
- PDF with 10k transactions: ~6-8s
- Timeout: 10s (return error if exceeds)

---

## Related Primitives

**Dependencies:**
- DashboardEngine (metrics are pre-calculated and passed in)
- TransactionQuery (transactions are queried separately)
- SavedViewStore (view config loaded before export)

**Used by:**
- API endpoint POST /api/dashboard/export

**Reuses:**
- ExportEngine from 2.1 (CSV export logic)
