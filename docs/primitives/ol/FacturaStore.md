# FacturaStore (OL Primitive)

**Vertical:** 3.4 Tax Categorization (Regulatory Classification)
**Layer:** Objective Layer (OL)
**Type:** Regulatory Document Store
**Status:** ✅ Specified

---

## Purpose

CRUD operations for Mexican Factura (CFDI) records with XML parsing, validation, transaction linking, and auto-matching. Provides comprehensive management of regulatory tax receipts required for compliance.

**Core Responsibility:** Persist and manage Mexican Factura (CFDI XML) records, parse CFDI 3.3 and 4.0 formats, validate structure and fields, link to transactions (1:1 or 1:N), support orphan Facturas (uploaded before transaction exists), and enable amount/date-based auto-matching.

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY regulatory document storage and linking:

| Domain | Document Type | Format | Fields | Linking |
|--------|---------------|--------|--------|---------|
| **Finance** | Mexican Factura (CFDI) | XML (CFDI 3.3/4.0) | RFC, UUID, amount, date | Link to transaction |
| **Healthcare** | CMS-1500 Claim Form | EDI 837, PDF | NPI, claim_id, amount, service_date | Link to patient visit |
| **Legal** | Court Filing | PDF, PACER XML | case_number, filing_date, court, document_type | Link to case |
| **Research** | Grant Budget Justification | PDF, Word | grant_id, budget_period, amount, institution | Link to grant |
| **Manufacturing** | Safety Inspection Report | PDF, XML | permit_id, inspection_date, inspector, status | Link to equipment |

**Pattern:** Regulatory Document Store = parse structured/unstructured documents → validate compliance → extract key fields → link to entities → support orphan docs (uploaded before entity exists) → enable auto-matching by amount/date.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List
from dataclasses import dataclass
from datetime import datetime, date
from decimal import Decimal
from enum import Enum

class FacturaStatus(Enum):
    """Factura validation status."""
    VALID = "valid"                    # All validations passed
    INVALID = "invalid"                # Validation failed
    PENDING_LINK = "pending_link"      # Valid but not linked to transaction yet
    ORPHAN = "orphan"                  # No matching transaction found

@dataclass
class FacturaRecord:
    """Represents a Mexican Factura (CFDI) record."""
    factura_id: str
    user_id: str
    transaction_id: Optional[str]  # None for orphan Facturas

    # CFDI fields
    rfc: str  # 12-13 character tax ID
    uuid: str  # UUID v4 format
    amount: Decimal
    currency: str  # MXN, USD, etc.
    issued_date: date

    # File references
    xml_file_id: str  # FileArtifact ID (from Vertical 1.1)
    pdf_file_id: Optional[str]  # Optional PDF representation

    # Validation
    status: FacturaStatus
    validation_errors: List[str]  # Empty if valid

    # Metadata
    created_at: datetime
    updated_at: datetime

@dataclass
class AutoMatchResult:
    """Result of auto-matching Factura to transaction."""
    factura: FacturaRecord
    matched_transaction: Optional['Transaction']
    confidence: float  # [0.0, 1.0]
    match_reason: str
    match_details: dict

class FacturaStore:
    """
    Manages CRUD operations for Mexican Factura (CFDI) records.

    Enforces:
    - Valid CFDI 3.3 or 4.0 XML format
    - RFC format validation (12-13 alphanumeric)
    - UUID format validation (UUIDv4)
    - Amount > 0
    - Date not in future
    - One-to-many linking (1 transaction can have N Facturas)
    - Orphan Factura support (uploaded before transaction exists)
    """

    def __init__(self, db_connection, storage_engine, factura_validator):
        self.db = db_connection
        self.storage_engine = storage_engine
        self.validator = factura_validator

    def create(
        self,
        user_id: str,
        xml_content: bytes,
        transaction_id: Optional[str] = None,
        pdf_content: Optional[bytes] = None
    ) -> FacturaRecord:
        """
        Parse CFDI XML, validate, and create Factura record.

        Process:
        1. Parse XML (CFDI 3.3 or 4.0)
        2. Validate structure (RFC, UUID, amount, date)
        3. Store XML file (via StorageEngine)
        4. Store PDF file (if provided)
        5. Link to transaction (if provided)
        6. Create FacturaRecord
        7. Return result

        Args:
            user_id: Owner of the Factura
            xml_content: CFDI XML file contents (bytes)
            transaction_id: Optional transaction to link (None = orphan)
            pdf_content: Optional PDF representation (bytes)

        Returns:
            Created FacturaRecord

        Raises:
            FacturaParseError: If XML is malformed or not CFDI format
            FacturaValidationError: If RFC/UUID/amount/date invalid
            TransactionMismatchError: If transaction amount doesn't match Factura (tolerance: ±5%)
            TransactionNotFoundError: If transaction_id provided but doesn't exist

        Example:
            # Upload Factura with transaction link
            with open("factura.xml", "rb") as f:
                xml_content = f.read()

            factura = store.create(
                user_id="user_darwin",
                xml_content=xml_content,
                transaction_id="txn_67890"
            )

            print(f"Factura created: {factura.factura_id}")
            print(f"RFC: {factura.rfc}")
            print(f"Amount: {factura.amount} {factura.currency}")
            print(f"Status: {factura.status}")

            # Upload orphan Factura (no transaction yet)
            orphan = store.create(
                user_id="user_darwin",
                xml_content=xml_content,
                transaction_id=None  # Will auto-match later
            )
            print(f"Orphan Factura: {orphan.factura_id}")
        """

    def get(self, factura_id: str, user_id: str) -> Optional[FacturaRecord]:
        """
        Get Factura by ID.

        Args:
            factura_id: Factura ID to fetch
            user_id: Owner (for authorization)

        Returns:
            FacturaRecord if found and user owns it, None otherwise

        Raises:
            UnauthorizedError: If Factura exists but user doesn't own it
        """

    def list(
        self,
        user_id: str,
        transaction_id: Optional[str] = None,
        status: Optional[FacturaStatus] = None,
        start_date: Optional[date] = None,
        end_date: Optional[date] = None
    ) -> List[FacturaRecord]:
        """
        List Facturas for user.

        Args:
            user_id: Owner of the Facturas
            transaction_id: Filter by transaction (optional)
            status: Filter by status (optional)
            start_date: Filter by issued_date >= start_date (optional)
            end_date: Filter by issued_date <= end_date (optional)

        Returns:
            List of FacturaRecord objects (sorted by issued_date DESC)

        Example:
            # Get all orphan Facturas (no transaction link)
            orphans = store.list(
                user_id="user_darwin",
                status=FacturaStatus.ORPHAN
            )

            # Get Facturas for specific transaction
            txn_facturas = store.list(
                user_id="user_darwin",
                transaction_id="txn_67890"
            )

            # Get Facturas for date range
            november_facturas = store.list(
                user_id="user_darwin",
                start_date=date(2024, 11, 1),
                end_date=date(2024, 11, 30)
            )
        """

    def link_to_transaction(
        self,
        factura_id: str,
        user_id: str,
        transaction_id: str,
        force: bool = False
    ) -> FacturaRecord:
        """
        Link Factura to transaction.

        Validates:
        - Amount matches (tolerance: ±5%)
        - Date within ±7 days
        - Currency matches (if transaction has currency)

        Args:
            factura_id: Factura to link
            user_id: Owner (for authorization)
            transaction_id: Transaction to link to
            force: Skip validation checks (force link even if mismatch)

        Returns:
            Updated FacturaRecord with transaction_id set

        Raises:
            FacturaNotFoundError: If factura_id doesn't exist
            TransactionNotFoundError: If transaction_id doesn't exist
            UnauthorizedError: If user doesn't own Factura or transaction
            TransactionMismatchError: If amount/date/currency don't match (unless force=True)
            AlreadyLinkedError: If Factura already linked to different transaction

        Example:
            # Link orphan Factura to transaction
            factura = store.link_to_transaction(
                factura_id="factura_abc123",
                user_id="user_darwin",
                transaction_id="txn_67890"
            )

            # Force link even if amounts don't match exactly
            factura = store.link_to_transaction(
                factura_id="factura_xyz789",
                user_id="user_darwin",
                transaction_id="txn_11111",
                force=True
            )
        """

    def auto_match(
        self,
        user_id: str,
        factura_id: Optional[str] = None,
        min_confidence: float = 0.80
    ) -> List[AutoMatchResult]:
        """
        Auto-match Facturas to transactions based on amount and date.

        Matching algorithm:
        1. Find transactions with matching amount (±5%)
        2. Filter by date within ±7 days
        3. Filter by currency (if available)
        4. Calculate confidence score
        5. Return matches above threshold

        Confidence scoring:
        - Exact amount + same date: 1.0
        - Amount within 1% + date within 1 day: 0.95
        - Amount within 5% + date within 3 days: 0.85
        - Amount within 5% + date within 7 days: 0.75

        Args:
            user_id: Owner of Facturas and transactions
            factura_id: Optional specific Factura to match (None = all orphans)
            min_confidence: Minimum confidence threshold (default: 0.80)

        Returns:
            List of AutoMatchResult objects (ordered by confidence DESC)

        Example:
            # Auto-match all orphan Facturas
            matches = store.auto_match(
                user_id="user_darwin",
                min_confidence=0.80
            )

            for result in matches:
                if result.confidence >= 0.90:
                    # Auto-apply high-confidence match
                    store.link_to_transaction(
                        factura_id=result.factura.factura_id,
                        user_id=user_id,
                        transaction_id=result.matched_transaction.transaction_id
                    )

            # Auto-match specific Factura
            matches = store.auto_match(
                user_id="user_darwin",
                factura_id="factura_abc123"
            )
        """

    def unlink_from_transaction(
        self,
        factura_id: str,
        user_id: str
    ) -> FacturaRecord:
        """
        Unlink Factura from transaction (make orphan again).

        Args:
            factura_id: Factura to unlink
            user_id: Owner (for authorization)

        Returns:
            Updated FacturaRecord with transaction_id = None, status = ORPHAN

        Example:
            # Unlink Factura if linked to wrong transaction
            factura = store.unlink_from_transaction(
                factura_id="factura_abc123",
                user_id="user_darwin"
            )
        """

    def download_xml(
        self,
        factura_id: str,
        user_id: str
    ) -> bytes:
        """
        Download original CFDI XML file.

        Args:
            factura_id: Factura ID
            user_id: Owner (for authorization)

        Returns:
            XML file contents (bytes)

        Example:
            xml_content = store.download_xml("factura_abc123", "user_darwin")
            with open("factura_backup.xml", "wb") as f:
                f.write(xml_content)
        """

    def download_pdf(
        self,
        factura_id: str,
        user_id: str
    ) -> Optional[bytes]:
        """
        Download PDF representation (if available).

        Args:
            factura_id: Factura ID
            user_id: Owner (for authorization)

        Returns:
            PDF file contents (bytes) or None if no PDF

        Example:
            pdf_content = store.download_pdf("factura_abc123", "user_darwin")
            if pdf_content:
                with open("factura.pdf", "wb") as f:
                    f.write(pdf_content)
        """

    def delete(
        self,
        factura_id: str,
        user_id: str
    ) -> None:
        """
        Delete Factura record and associated files.

        Hard delete (not soft delete) since Factura is external artifact.

        Args:
            factura_id: Factura to delete
            user_id: Owner (for authorization)

        Raises:
            FacturaNotFoundError: If factura_id doesn't exist
            UnauthorizedError: If user doesn't own Factura
        """

    def _generate_factura_id(self, rfc: str) -> str:
        """
        Generate deterministic factura_id.

        Format: factura_{rfc}_{sequence}
        Example: "ABC123456XYZ" → "factura_abc123456xyz_1"

        Args:
            rfc: RFC from CFDI

        Returns:
            factura_id string
        """
```

---

## Implementation Details

### Database Schema

```sql
CREATE TABLE facturas (
    factura_id VARCHAR(100) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,
    transaction_id VARCHAR(100),                -- NULL for orphan Facturas

    -- CFDI fields
    rfc VARCHAR(13) NOT NULL,
    uuid UUID NOT NULL,
    amount DECIMAL(12,2) NOT NULL CHECK (amount > 0),
    currency VARCHAR(3) NOT NULL,               -- MXN, USD, etc.
    issued_date DATE NOT NULL CHECK (issued_date <= CURRENT_DATE),

    -- File references
    xml_file_id VARCHAR(100) NOT NULL,          -- References file_artifacts table
    pdf_file_id VARCHAR(100),                   -- Optional PDF

    -- Validation
    status VARCHAR(20) NOT NULL,                -- valid, invalid, pending_link, orphan
    validation_errors JSONB,                    -- Array of error messages

    -- Metadata
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,

    -- Constraints
    INDEX idx_user (user_id),
    INDEX idx_transaction (transaction_id),
    INDEX idx_rfc (rfc),
    INDEX idx_uuid (uuid),
    INDEX idx_issued_date (issued_date),
    INDEX idx_status (status),
    INDEX idx_orphans (user_id, status) WHERE status = 'orphan',
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (transaction_id) REFERENCES transactions(transaction_id),
    FOREIGN KEY (xml_file_id) REFERENCES file_artifacts(artifact_id),
    FOREIGN KEY (pdf_file_id) REFERENCES file_artifacts(artifact_id)
);

-- Change log for audit trail
CREATE TABLE factura_change_log (
    log_id VARCHAR(50) PRIMARY KEY,
    factura_id VARCHAR(100) NOT NULL,
    user_id VARCHAR(50) NOT NULL,

    operation VARCHAR(20) NOT NULL,             -- CREATE, LINK, UNLINK, DELETE
    changes JSONB,                              -- {field: {old: X, new: Y}}
    timestamp TIMESTAMP NOT NULL,

    FOREIGN KEY (factura_id) REFERENCES facturas(factura_id),
    INDEX idx_factura (factura_id),
    INDEX idx_user (user_id)
);
```

### Factura ID Generation

```python
def _generate_factura_id(self, rfc: str) -> str:
    """
    Generate factura_id: factura_{rfc}_{sequence}

    Examples:
      "ABC123456XYZ" → "factura_abc123456xyz_1"
      "XAXX010101000" → "factura_xaxx010101000_1"
    """
    # Normalize RFC
    rfc_normalized = rfc.lower()

    # Find next sequence number
    existing = self.db.execute(
        "SELECT factura_id FROM facturas WHERE factura_id LIKE ? ORDER BY factura_id DESC LIMIT 1",
        (f"factura_{rfc_normalized}_%",)
    ).fetchone()

    if existing:
        # Extract sequence from "factura_abc123456xyz_3" → 3
        last_seq = int(existing['factura_id'].split('_')[-1])
        seq = last_seq + 1
    else:
        seq = 1

    return f"factura_{rfc_normalized}_{seq}"
```

### XML Parsing (CFDI 3.3 and 4.0)

```python
def _parse_cfdi_xml(self, xml_content: bytes) -> dict:
    """
    Parse CFDI XML (3.3 or 4.0 format).

    Extracts:
    - RFC (Receptor.Rfc)
    - UUID (TimbreFiscalDigital.UUID)
    - Amount (Total)
    - Currency (Moneda)
    - Date (Fecha)

    Returns:
        Dictionary with extracted fields

    Raises:
        FacturaParseError: If XML malformed or missing required fields
    """
    import xml.etree.ElementTree as ET

    try:
        root = ET.fromstring(xml_content)
    except ET.ParseError as e:
        raise FacturaParseError(f"Invalid XML format: {e}")

    # Determine CFDI version
    version = root.get('Version')
    if version not in ['3.3', '4.0']:
        raise FacturaParseError(f"Unsupported CFDI version: {version}")

    # Define namespaces
    if version == '3.3':
        ns = {
            'cfdi': 'http://www.sat.gob.mx/cfd/3',
            'tfd': 'http://www.sat.gob.mx/TimbreFiscalDigital'
        }
    else:  # 4.0
        ns = {
            'cfdi': 'http://www.sat.gob.mx/cfd/4',
            'tfd': 'http://www.sat.gob.mx/TimbreFiscalDigital'
        }

    # Extract fields
    try:
        # RFC (Receptor)
        receptor = root.find('cfdi:Receptor', ns)
        if receptor is None:
            raise FacturaParseError("Missing Receptor element")
        rfc = receptor.get('Rfc')

        # UUID (TimbreFiscalDigital)
        complemento = root.find('cfdi:Complemento', ns)
        if complemento is None:
            raise FacturaParseError("Missing Complemento element")
        tfd = complemento.find('tfd:TimbreFiscalDigital', ns)
        if tfd is None:
            raise FacturaParseError("Missing TimbreFiscalDigital element")
        uuid = tfd.get('UUID')

        # Amount (Total)
        amount = Decimal(root.get('Total'))

        # Currency (Moneda)
        currency = root.get('Moneda') or 'MXN'

        # Date (Fecha)
        fecha_str = root.get('Fecha')
        # Parse ISO format: "2024-11-01T10:30:00"
        issued_date = datetime.fromisoformat(fecha_str.split('T')[0]).date()

        return {
            'rfc': rfc,
            'uuid': uuid,
            'amount': amount,
            'currency': currency,
            'issued_date': issued_date,
            'version': version
        }

    except (AttributeError, KeyError, ValueError) as e:
        raise FacturaParseError(f"Error extracting CFDI fields: {e}")
```

### Transaction Matching Validation

```python
def _validate_transaction_match(
    self,
    factura_amount: Decimal,
    factura_date: date,
    factura_currency: str,
    transaction: 'Transaction'
) -> tuple[bool, List[str]]:
    """
    Validate Factura matches transaction.

    Checks:
    1. Amount within ±5%
    2. Date within ±7 days
    3. Currency matches (if transaction has currency)

    Returns:
        (is_valid, error_messages)
    """
    errors = []

    # Check amount (±5% tolerance)
    txn_amount = abs(transaction.amount)
    variance = abs(factura_amount - txn_amount)
    tolerance_pct = 0.05
    tolerance_amount = txn_amount * Decimal(str(tolerance_pct))

    if variance > tolerance_amount:
        variance_pct = (variance / txn_amount) * 100
        errors.append(
            f"Amount mismatch: Factura ${factura_amount}, Transaction ${txn_amount} "
            f"(variance: {variance_pct:.1f}%, tolerance: {tolerance_pct * 100}%)"
        )

    # Check date (±7 days)
    date_diff = abs((factura_date - transaction.date).days)
    if date_diff > 7:
        errors.append(
            f"Date mismatch: Factura {factura_date}, Transaction {transaction.date} "
            f"(difference: {date_diff} days, tolerance: 7 days)"
        )

    # Check currency (if available)
    if hasattr(transaction, 'currency') and transaction.currency:
        if factura_currency != transaction.currency:
            errors.append(
                f"Currency mismatch: Factura {factura_currency}, Transaction {transaction.currency}"
            )

    return (len(errors) == 0, errors)
```

### Auto-Matching Algorithm

```python
def auto_match(
    self,
    user_id: str,
    factura_id: Optional[str] = None,
    min_confidence: float = 0.80
) -> List[AutoMatchResult]:
    """
    Auto-match Facturas to transactions.

    Algorithm:
    1. Get orphan Facturas (or specific Factura)
    2. For each Factura:
       a. Find transactions with amount in range [amount * 0.95, amount * 1.05]
       b. Filter by date within ±7 days
       c. Calculate confidence score
       d. Return if >= min_confidence
    """
    # Get Facturas to match
    if factura_id:
        facturas = [self.get(factura_id, user_id)]
    else:
        facturas = self.list(user_id=user_id, status=FacturaStatus.ORPHAN)

    results = []

    for factura in facturas:
        # Find candidate transactions
        amount_min = factura.amount * Decimal('0.95')
        amount_max = factura.amount * Decimal('1.05')

        candidates = self.transaction_store.list(
            user_id=user_id,
            amount_min=-amount_max,  # Negative for expenses
            amount_max=-amount_min,
            date_min=factura.issued_date - timedelta(days=7),
            date_max=factura.issued_date + timedelta(days=7),
            has_factura=False  # Only unlinked transactions
        )

        # Score each candidate
        for txn in candidates:
            confidence, reason, details = self._calculate_match_confidence(
                factura, txn
            )

            if confidence >= min_confidence:
                results.append(AutoMatchResult(
                    factura=factura,
                    matched_transaction=txn,
                    confidence=confidence,
                    match_reason=reason,
                    match_details=details
                ))

    # Sort by confidence DESC
    results.sort(key=lambda r: r.confidence, reverse=True)
    return results

def _calculate_match_confidence(
    self,
    factura: FacturaRecord,
    transaction: 'Transaction'
) -> tuple[float, str, dict]:
    """
    Calculate match confidence score.

    Scoring:
    - Exact amount + same date: 1.0
    - Amount within 1% + date within 1 day: 0.95
    - Amount within 2% + date within 2 days: 0.90
    - Amount within 5% + date within 3 days: 0.85
    - Amount within 5% + date within 7 days: 0.75
    """
    txn_amount = abs(transaction.amount)
    amount_diff_pct = abs(factura.amount - txn_amount) / txn_amount
    date_diff_days = abs((factura.issued_date - transaction.date).days)

    # Exact match
    if amount_diff_pct < 0.001 and date_diff_days == 0:
        return (1.0, "Exact amount and date match", {
            'amount_diff_pct': float(amount_diff_pct),
            'date_diff_days': date_diff_days
        })

    # Very close match
    if amount_diff_pct <= 0.01 and date_diff_days <= 1:
        return (0.95, "Amount within 1% and date within 1 day", {
            'amount_diff_pct': float(amount_diff_pct),
            'date_diff_days': date_diff_days
        })

    # Close match
    if amount_diff_pct <= 0.02 and date_diff_days <= 2:
        return (0.90, "Amount within 2% and date within 2 days", {
            'amount_diff_pct': float(amount_diff_pct),
            'date_diff_days': date_diff_days
        })

    # Good match
    if amount_diff_pct <= 0.05 and date_diff_days <= 3:
        return (0.85, "Amount within 5% and date within 3 days", {
            'amount_diff_pct': float(amount_diff_pct),
            'date_diff_days': date_diff_days
        })

    # Acceptable match
    if amount_diff_pct <= 0.05 and date_diff_days <= 7:
        return (0.75, "Amount within 5% and date within 7 days", {
            'amount_diff_pct': float(amount_diff_pct),
            'date_diff_days': date_diff_days
        })

    # No match
    return (0.0, "Match criteria not met", {
        'amount_diff_pct': float(amount_diff_pct),
        'date_diff_days': date_diff_days
    })
```

---

## Usage Examples

### Example 1: Upload Factura with Transaction Link

```python
store = FacturaStore(db_connection, storage_engine, factura_validator)

# Read XML file
with open("factura.xml", "rb") as f:
    xml_content = f.read()

# Create Factura linked to transaction
factura = store.create(
    user_id="user_darwin",
    xml_content=xml_content,
    transaction_id="txn_67890"
)

print(f"Factura created: {factura.factura_id}")
print(f"RFC: {factura.rfc}")
print(f"UUID: {factura.uuid}")
print(f"Amount: ${factura.amount} {factura.currency}")
print(f"Date: {factura.issued_date}")
print(f"Status: {factura.status.value}")
```

### Example 2: Upload Orphan Factura (No Transaction Yet)

```python
# Upload Factura before transaction is imported
orphan = store.create(
    user_id="user_darwin",
    xml_content=xml_content,
    transaction_id=None  # No link yet
)

print(f"Orphan Factura: {orphan.factura_id}")
print(f"Status: {orphan.status.value}")  # "orphan"
```

### Example 3: Auto-Match Orphan Facturas

```python
# Auto-match all orphan Facturas
matches = store.auto_match(
    user_id="user_darwin",
    min_confidence=0.80
)

print(f"Found {len(matches)} potential matches")

for result in matches:
    print(f"\nFactura: {result.factura.factura_id}")
    print(f"  Transaction: {result.matched_transaction.transaction_id}")
    print(f"  Confidence: {result.confidence:.0%}")
    print(f"  Reason: {result.match_reason}")

    if result.confidence >= 0.90:
        # Auto-apply high-confidence match
        store.link_to_transaction(
            factura_id=result.factura.factura_id,
            user_id=user_id,
            transaction_id=result.matched_transaction.transaction_id
        )
        print("  → Auto-linked")
```

### Example 4: Manual Link Factura to Transaction

```python
# User manually links Factura
factura = store.link_to_transaction(
    factura_id="factura_abc123456xyz_1",
    user_id="user_darwin",
    transaction_id="txn_67890"
)

print(f"Linked Factura to transaction")
print(f"Status: {factura.status.value}")  # "valid"
```

### Example 5: Force Link Despite Mismatch

```python
# Amounts don't match exactly (8% variance)
try:
    factura = store.link_to_transaction(
        factura_id="factura_xyz789",
        user_id="user_darwin",
        transaction_id="txn_11111"
    )
except TransactionMismatchError as e:
    print(f"Mismatch detected: {e.errors}")

    # User confirms: force link anyway
    factura = store.link_to_transaction(
        factura_id="factura_xyz789",
        user_id="user_darwin",
        transaction_id="txn_11111",
        force=True
    )
    print("Force-linked despite mismatch")
```

### Example 6: Download XML and PDF

```python
# Download original XML
xml_content = store.download_xml("factura_abc123", "user_darwin")
with open("factura_backup.xml", "wb") as f:
    f.write(xml_content)

# Download PDF (if available)
pdf_content = store.download_pdf("factura_abc123", "user_darwin")
if pdf_content:
    with open("factura.pdf", "wb") as f:
        f.write(pdf_content)
```

### Example 7: List Orphan Facturas

```python
# Get all Facturas without transaction link
orphans = store.list(
    user_id="user_darwin",
    status=FacturaStatus.ORPHAN
)

print(f"Found {len(orphans)} orphan Facturas:")
for factura in orphans:
    print(f"  {factura.rfc} - ${factura.amount} - {factura.issued_date}")
```

---

## Multi-Domain Examples

### Healthcare: CMS-1500 Claim Form Store

```python
class ClaimFormStore:
    """
    Store CMS-1500 claim forms (Healthcare).

    Similar pattern to FacturaStore:
    - Parse EDI 837 format
    - Validate NPI, claim_id, amounts
    - Link to patient visit
    - Support orphan claims
    - Auto-match by amount + service_date
    """

    def create(self, user_id, edi_content, visit_id=None):
        # Parse EDI 837
        # Validate NPI, claim_id
        # Store claim record
        # Link to visit (if provided)
        pass
```

### Legal: Court Filing Store

```python
class CourtFilingStore:
    """
    Store court filings (Legal).

    Similar pattern:
    - Parse PACER XML or PDF
    - Validate case_number, filing_date, court
    - Link to case
    - Support orphan filings
    - Auto-match by case_number + filing_date
    """

    def create(self, user_id, filing_content, case_id=None):
        # Parse PACER XML
        # Validate case_number
        # Store filing record
        # Link to case (if provided)
        pass
```

### Research: Grant Budget Justification Store

```python
class BudgetJustificationStore:
    """
    Store grant budget justifications (Research).

    Similar pattern:
    - Parse PDF or Word document
    - Extract grant_id, budget_period, amounts
    - Link to grant
    - Support orphan budgets
    - Auto-match by grant_id + budget_period
    """

    def create(self, user_id, document_content, grant_id=None):
        # Parse PDF/Word
        # Extract fields
        # Store budget record
        # Link to grant (if provided)
        pass
```

---

## Error Handling

### Custom Exceptions

```python
class FacturaStoreError(Exception):
    """Base exception for FacturaStore errors."""
    pass

class FacturaParseError(FacturaStoreError):
    def __init__(self, message: str):
        self.message = message
        super().__init__(message)

class FacturaValidationError(FacturaStoreError):
    def __init__(self, message: str, errors: List[str]):
        self.message = message
        self.errors = errors
        super().__init__(message)

class TransactionMismatchError(FacturaStoreError):
    def __init__(self, message: str, errors: List[str]):
        self.message = message
        self.errors = errors
        super().__init__(message)

class TransactionNotFoundError(FacturaStoreError):
    pass

class FacturaNotFoundError(FacturaStoreError):
    pass

class UnauthorizedError(FacturaStoreError):
    pass

class AlreadyLinkedError(FacturaStoreError):
    def __init__(self, message: str, linked_transaction_id: str):
        self.message = message
        self.linked_transaction_id = linked_transaction_id
        super().__init__(message)
```

---

## Performance Characteristics

### Query Performance

| Operation | Expected Latency (p95) | Optimization |
|-----------|------------------------|--------------|
| `create()` | <500ms | XML parsing + file storage |
| `get()` | <20ms | Indexed by PK |
| `list()` orphans | <50ms | Indexed by (user_id, status) |
| `link_to_transaction()` | <100ms | Validation + update |
| `auto_match()` (100 orphans) | <2000ms | Batch queries |

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Create Factura
def test_create_factura_with_transaction()
def test_create_factura_orphan()
def test_create_factura_invalid_xml()
def test_create_factura_invalid_rfc()
def test_create_factura_amount_mismatch()

# Test: Link to transaction
def test_link_to_transaction_success()
def test_link_to_transaction_amount_mismatch()
def test_link_to_transaction_force()
def test_link_to_transaction_already_linked()

# Test: Auto-match
def test_auto_match_exact()
def test_auto_match_close()
def test_auto_match_no_match()
def test_auto_match_confidence_scoring()

# Test: Download
def test_download_xml()
def test_download_pdf()
def test_download_unauthorized()
```

---

## Related Primitives

- **FacturaValidator** (OL) - Validates CFDI XML format
- **StorageEngine** (from Vertical 1.1) - Stores XML and PDF files
- **TransactionStore** (from Vertical 1.3) - Links to transactions
- **FacturaUploadDialog** (IL) - UI for uploading Facturas

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 3-4 days
**Dependencies:** FacturaValidator, StorageEngine, TransactionStore, XML parser (lxml or ElementTree)
