# FacturaValidator (OL Primitive)

**Vertical:** 3.4 Tax Categorization (Regulatory Classification)
**Layer:** Objective Layer (OL)
**Type:** Regulatory Document Validator
**Status:** ✅ Specified

---

## Purpose

Validate CFDI (Comprobante Fiscal Digital por Internet) XML format for Mexican tax compliance. Provides comprehensive validation of structure, format, and business rules for CFDI 3.3 and 4.0 standards.

**Core Responsibility:** Validate Mexican Factura XML files against SAT (Servicio de Administración Tributaria) schemas, enforce RFC format rules, validate UUID format, verify amount constraints, check date validity, and optionally verify digital signatures and SAT database lookups.

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY regulatory document validation:

| Domain | Document Type | Validation Rules | Standards |
|--------|---------------|------------------|-----------|
| **Finance** | Mexican Factura (CFDI) | RFC format, UUID format, XML schema, digital signature | SAT CFDI 3.3/4.0 |
| **Healthcare** | HL7 Message | Segment structure, field types, value sets, checksums | HL7 v2.x, v3, FHIR |
| **Legal** | Legal XML (ECFS) | Case number format, filing date, document type, schema | US Courts PACER |
| **Research** | Grant Submission XML | Grant ID format, budget schema, institution validation | NSF, NIH FastLane |
| **Manufacturing** | Safety Inspection XML | Permit ID, inspector credentials, compliance status | OSHA, EPA standards |

**Pattern:** Regulatory Document Validator = parse structured document → validate schema compliance → check field formats → verify business rules → optionally verify with external authority → return validation result with detailed errors.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List
from dataclasses import dataclass
from datetime import date
from decimal import Decimal
from enum import Enum

class ValidationSeverity(Enum):
    """Validation error severity levels."""
    ERROR = "error"        # Must fix (blocks processing)
    WARNING = "warning"    # Should fix (continues processing)
    INFO = "info"          # Informational only

@dataclass
class ValidationError:
    """Represents a validation error."""
    field: str                    # Field that failed validation
    message: str                  # Human-readable error message
    severity: ValidationSeverity
    expected: Optional[str] = None  # Expected value/format
    actual: Optional[str] = None    # Actual value found
    rule: Optional[str] = None      # Validation rule ID

@dataclass
class ValidationResult:
    """Result of validation operation."""
    is_valid: bool
    errors: List[ValidationError]  # All errors (ERROR, WARNING, INFO)
    warnings: List[ValidationError]  # WARNING level only
    version: Optional[str] = None  # CFDI version (3.3 or 4.0)

    def has_errors(self) -> bool:
        """Check if any ERROR-level issues exist."""
        return any(e.severity == ValidationSeverity.ERROR for e in self.errors)

    def get_errors(self) -> List[ValidationError]:
        """Get ERROR-level issues only."""
        return [e for e in self.errors if e.severity == ValidationSeverity.ERROR]

    def get_warnings(self) -> List[ValidationError]:
        """Get WARNING-level issues only."""
        return [e for e in self.errors if e.severity == ValidationSeverity.WARNING]

class FacturaValidator:
    """
    Validate CFDI XML format for Mexican tax compliance.

    Validation levels:
    1. Structure: Valid XML, correct namespaces
    2. Schema: Compliance with SAT XSD schema
    3. Format: RFC, UUID, amount, date formats
    4. Business Rules: Amount > 0, date not in future, etc.
    5. Optional: Digital signature, SAT database lookup
    """

    def __init__(self, sat_api_client=None):
        self.sat_api_client = sat_api_client  # Optional SAT API for UUID verification

    def validate_xml(
        self,
        xml_content: bytes,
        validate_signature: bool = False,
        verify_sat_database: bool = False
    ) -> ValidationResult:
        """
        Validate CFDI XML format.

        Validation steps:
        1. Parse XML (check well-formed)
        2. Detect CFDI version (3.3 or 4.0)
        3. Validate against XSD schema
        4. Validate field formats (RFC, UUID, amount, date)
        5. Validate business rules
        6. Optional: Validate digital signature
        7. Optional: Verify UUID in SAT database

        Args:
            xml_content: CFDI XML file contents (bytes)
            validate_signature: Verify digital signature (default: False)
            verify_sat_database: Check UUID against SAT database (default: False)

        Returns:
            ValidationResult with is_valid flag and error list

        Example:
            validator = FacturaValidator()

            with open("factura.xml", "rb") as f:
                xml_content = f.read()

            result = validator.validate_xml(xml_content)

            if result.is_valid:
                print("✓ Factura is valid")
            else:
                print("✗ Validation failed:")
                for error in result.get_errors():
                    print(f"  - {error.field}: {error.message}")
        """

    def validate_rfc(self, rfc: str) -> ValidationResult:
        """
        Validate RFC (Registro Federal de Contribuyentes) format.

        RFC rules:
        - Person (física): 13 characters (AAAA######XXX)
          - 4 letters (first surname + first name initial)
          - 6 digits (birth date YYMMDD)
          - 3 alphanumeric (homoclave)
        - Company (moral): 12 characters (AAA######XXX)
          - 3 letters (company name)
          - 6 digits (registration date YYMMDD)
          - 3 alphanumeric (homoclave)

        Args:
            rfc: RFC string to validate

        Returns:
            ValidationResult (is_valid, errors)

        Example:
            result = validator.validate_rfc("XAXX010101000")
            if not result.is_valid:
                print(f"Invalid RFC: {result.errors[0].message}")
        """

    def validate_uuid(self, uuid: str) -> ValidationResult:
        """
        Validate UUID format (UUIDv4).

        UUID format: 8-4-4-4-12 hexadecimal
        Example: 12345678-1234-1234-1234-123456789012

        Args:
            uuid: UUID string to validate

        Returns:
            ValidationResult (is_valid, errors)

        Example:
            result = validator.validate_uuid("12345678-1234-1234-1234-123456789012")
        """

    def validate_amount(self, amount: Decimal) -> ValidationResult:
        """
        Validate amount constraints.

        Rules:
        - Must be > 0
        - Must have max 2 decimal places
        - Must be < 1,000,000,000 (practical limit)

        Args:
            amount: Amount to validate

        Returns:
            ValidationResult (is_valid, errors)

        Example:
            result = validator.validate_amount(Decimal("500.00"))
        """

    def validate_date(self, date_value: date) -> ValidationResult:
        """
        Validate date constraints.

        Rules:
        - Must not be in the future
        - Must not be before 2000-01-01 (SAT CFDI inception)

        Args:
            date_value: Date to validate

        Returns:
            ValidationResult (is_valid, errors)

        Example:
            result = validator.validate_date(date(2024, 11, 1))
        """

    def verify_signature(self, xml_content: bytes) -> ValidationResult:
        """
        Verify CFDI digital signature (Sello Digital).

        Validates:
        - Signature element exists
        - Certificate chain is valid
        - Signature matches content

        Args:
            xml_content: CFDI XML file contents

        Returns:
            ValidationResult (is_valid, errors)

        Note: Requires cryptography library for signature verification
        """

    def verify_sat_database(self, uuid: str) -> ValidationResult:
        """
        Verify UUID exists in SAT database (external API call).

        Queries SAT web service to confirm UUID is registered.

        Args:
            uuid: UUID to verify

        Returns:
            ValidationResult (is_valid, errors)

        Raises:
            SATAPIError: If SAT API is unavailable

        Note: Requires SAT API credentials and network access
        """

    def _validate_schema(
        self,
        xml_content: bytes,
        version: str
    ) -> List[ValidationError]:
        """
        Validate XML against SAT XSD schema.

        Internal method used by validate_xml().

        Args:
            xml_content: CFDI XML content
            version: CFDI version (3.3 or 4.0)

        Returns:
            List of ValidationError objects
        """

    def _validate_business_rules(
        self,
        parsed_data: dict
    ) -> List[ValidationError]:
        """
        Validate business rules.

        Rules:
        - Total > 0
        - SubTotal >= 0
        - Total = SubTotal - Descuento + Impuestos
        - If IVA: rate in [0%, 8%, 16%]
        - Receptor.Rfc != Emisor.Rfc (can't invoice yourself)

        Internal method used by validate_xml().

        Args:
            parsed_data: Parsed CFDI fields

        Returns:
            List of ValidationError objects
        """
```

---

## Implementation Details

### RFC Validation

```python
def validate_rfc(self, rfc: str) -> ValidationResult:
    """
    Validate RFC format.

    RFC patterns:
    - Person: 13 chars (e.g., XAXX010101000)
    - Company: 12 chars (e.g., XAX0101010X0)

    Validation:
    1. Length check (12 or 13)
    2. Character pattern check (alphanumeric)
    3. Optional: Checksum validation (homoclave)
    """
    import re

    errors = []

    # Check length
    if len(rfc) not in [12, 13]:
        errors.append(ValidationError(
            field="rfc",
            message=f"RFC must be 12 or 13 characters, found {len(rfc)}",
            severity=ValidationSeverity.ERROR,
            expected="12 or 13 characters",
            actual=str(len(rfc)),
            rule="RFC_LENGTH"
        ))
        return ValidationResult(is_valid=False, errors=errors, warnings=[])

    # Check alphanumeric
    if not re.match(r'^[A-Z&Ñ]{3,4}\d{6}[A-Z0-9]{3}$', rfc, re.IGNORECASE):
        errors.append(ValidationError(
            field="rfc",
            message="RFC format invalid (must match pattern: AAA######XXX or AAAA######XXX)",
            severity=ValidationSeverity.ERROR,
            expected="Pattern: [A-Z]{3,4}[0-9]{6}[A-Z0-9]{3}",
            actual=rfc,
            rule="RFC_PATTERN"
        ))
        return ValidationResult(is_valid=False, errors=errors, warnings=[])

    # Check for test RFC (generic)
    if rfc == "XAXX010101000" or rfc == "XEXX010101000":
        errors.append(ValidationError(
            field="rfc",
            message="RFC is a test/generic value",
            severity=ValidationSeverity.WARNING,
            actual=rfc,
            rule="RFC_GENERIC"
        ))

    is_valid = len([e for e in errors if e.severity == ValidationSeverity.ERROR]) == 0
    return ValidationResult(is_valid=is_valid, errors=errors, warnings=[e for e in errors if e.severity == ValidationSeverity.WARNING])
```

### UUID Validation

```python
def validate_uuid(self, uuid: str) -> ValidationResult:
    """
    Validate UUID format (UUIDv4).

    Format: 8-4-4-4-12 hexadecimal
    Example: 12345678-1234-4234-a234-123456789012
              (note: version 4 has '4' in third group first digit)
    """
    import re
    import uuid as uuid_lib

    errors = []

    # Check format
    uuid_pattern = r'^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$'
    if not re.match(uuid_pattern, uuid, re.IGNORECASE):
        errors.append(ValidationError(
            field="uuid",
            message="UUID format invalid",
            severity=ValidationSeverity.ERROR,
            expected="Format: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx (hexadecimal)",
            actual=uuid,
            rule="UUID_FORMAT"
        ))
        return ValidationResult(is_valid=False, errors=errors, warnings=[])

    # Validate using Python uuid library
    try:
        uuid_obj = uuid_lib.UUID(uuid)
        # Check if version 4 (SAT uses UUIDv4)
        if uuid_obj.version != 4:
            errors.append(ValidationError(
                field="uuid",
                message=f"UUID must be version 4, found version {uuid_obj.version}",
                severity=ValidationSeverity.WARNING,
                expected="UUIDv4",
                actual=f"UUID v{uuid_obj.version}",
                rule="UUID_VERSION"
            ))
    except ValueError as e:
        errors.append(ValidationError(
            field="uuid",
            message=f"Invalid UUID: {e}",
            severity=ValidationSeverity.ERROR,
            rule="UUID_INVALID"
        ))
        return ValidationResult(is_valid=False, errors=errors, warnings=[])

    is_valid = len([e for e in errors if e.severity == ValidationSeverity.ERROR]) == 0
    return ValidationResult(is_valid=is_valid, errors=errors, warnings=[e for e in errors if e.severity == ValidationSeverity.WARNING])
```

### Amount Validation

```python
def validate_amount(self, amount: Decimal) -> ValidationResult:
    """
    Validate amount constraints.

    Rules:
    1. Must be > 0
    2. Max 2 decimal places
    3. Practical max: 1,000,000,000
    """
    errors = []

    # Check positive
    if amount <= 0:
        errors.append(ValidationError(
            field="amount",
            message="Amount must be greater than 0",
            severity=ValidationSeverity.ERROR,
            expected="> 0",
            actual=str(amount),
            rule="AMOUNT_POSITIVE"
        ))

    # Check decimal places
    if amount.as_tuple().exponent < -2:
        errors.append(ValidationError(
            field="amount",
            message="Amount must have maximum 2 decimal places",
            severity=ValidationSeverity.ERROR,
            expected="Max 2 decimal places",
            actual=f"{-amount.as_tuple().exponent} decimal places",
            rule="AMOUNT_DECIMALS"
        ))

    # Check practical maximum
    max_amount = Decimal("1000000000")  # 1 billion
    if amount > max_amount:
        errors.append(ValidationError(
            field="amount",
            message=f"Amount exceeds practical maximum ({max_amount})",
            severity=ValidationSeverity.WARNING,
            expected=f"<= {max_amount}",
            actual=str(amount),
            rule="AMOUNT_MAX"
        ))

    is_valid = len([e for e in errors if e.severity == ValidationSeverity.ERROR]) == 0
    return ValidationResult(is_valid=is_valid, errors=errors, warnings=[e for e in errors if e.severity == ValidationSeverity.WARNING])
```

### Date Validation

```python
def validate_date(self, date_value: date) -> ValidationResult:
    """
    Validate date constraints.

    Rules:
    1. Not in future
    2. Not before 2000-01-01 (SAT CFDI inception)
    """
    from datetime import date as date_class

    errors = []

    today = date_class.today()

    # Check future
    if date_value > today:
        errors.append(ValidationError(
            field="date",
            message="Date cannot be in the future",
            severity=ValidationSeverity.ERROR,
            expected=f"<= {today}",
            actual=str(date_value),
            rule="DATE_FUTURE"
        ))

    # Check too old (CFDI started 2004, but 2000 is safe lower bound)
    min_date = date_class(2000, 1, 1)
    if date_value < min_date:
        errors.append(ValidationError(
            field="date",
            message=f"Date too old (CFDI system started after {min_date})",
            severity=ValidationSeverity.WARNING,
            expected=f">= {min_date}",
            actual=str(date_value),
            rule="DATE_TOO_OLD"
        ))

    is_valid = len([e for e in errors if e.severity == ValidationSeverity.ERROR]) == 0
    return ValidationResult(is_valid=is_valid, errors=errors, warnings=[e for e in errors if e.severity == ValidationSeverity.WARNING])
```

### Full XML Validation

```python
def validate_xml(
    self,
    xml_content: bytes,
    validate_signature: bool = False,
    verify_sat_database: bool = False
) -> ValidationResult:
    """
    Comprehensive CFDI XML validation.

    Steps:
    1. Parse XML
    2. Detect version
    3. Validate schema
    4. Validate field formats
    5. Validate business rules
    6. Optional: signature
    7. Optional: SAT database
    """
    import xml.etree.ElementTree as ET

    all_errors = []

    # Step 1: Parse XML
    try:
        root = ET.fromstring(xml_content)
    except ET.ParseError as e:
        all_errors.append(ValidationError(
            field="xml",
            message=f"Invalid XML format: {e}",
            severity=ValidationSeverity.ERROR,
            rule="XML_PARSE"
        ))
        return ValidationResult(is_valid=False, errors=all_errors, warnings=[])

    # Step 2: Detect version
    version = root.get('Version')
    if version not in ['3.3', '4.0']:
        all_errors.append(ValidationError(
            field="version",
            message=f"Unsupported CFDI version: {version}",
            severity=ValidationSeverity.ERROR,
            expected="3.3 or 4.0",
            actual=version,
            rule="VERSION_UNSUPPORTED"
        ))
        return ValidationResult(is_valid=False, errors=all_errors, warnings=[], version=version)

    # Step 3: Validate schema (XSD)
    schema_errors = self._validate_schema(xml_content, version)
    all_errors.extend(schema_errors)

    # Step 4: Extract and validate fields
    try:
        parsed_data = self._parse_cfdi_fields(root, version)

        # Validate RFC
        rfc_result = self.validate_rfc(parsed_data['rfc'])
        all_errors.extend(rfc_result.errors)

        # Validate UUID
        uuid_result = self.validate_uuid(parsed_data['uuid'])
        all_errors.extend(uuid_result.errors)

        # Validate amount
        amount_result = self.validate_amount(parsed_data['total'])
        all_errors.extend(amount_result.errors)

        # Validate date
        date_result = self.validate_date(parsed_data['fecha'])
        all_errors.extend(date_result.errors)

        # Step 5: Business rules
        business_errors = self._validate_business_rules(parsed_data)
        all_errors.extend(business_errors)

    except Exception as e:
        all_errors.append(ValidationError(
            field="cfdi",
            message=f"Error extracting CFDI fields: {e}",
            severity=ValidationSeverity.ERROR,
            rule="CFDI_EXTRACT"
        ))
        return ValidationResult(is_valid=False, errors=all_errors, warnings=[], version=version)

    # Step 6: Optional signature validation
    if validate_signature:
        sig_result = self.verify_signature(xml_content)
        all_errors.extend(sig_result.errors)

    # Step 7: Optional SAT database verification
    if verify_sat_database and parsed_data.get('uuid'):
        try:
            sat_result = self.verify_sat_database(parsed_data['uuid'])
            all_errors.extend(sat_result.errors)
        except SATAPIError as e:
            all_errors.append(ValidationError(
                field="uuid",
                message=f"SAT database verification failed: {e}",
                severity=ValidationSeverity.WARNING,
                rule="SAT_API_ERROR"
            ))

    # Determine if valid
    is_valid = len([e for e in all_errors if e.severity == ValidationSeverity.ERROR]) == 0

    return ValidationResult(
        is_valid=is_valid,
        errors=all_errors,
        warnings=[e for e in all_errors if e.severity == ValidationSeverity.WARNING],
        version=version
    )
```

### Business Rules Validation

```python
def _validate_business_rules(self, parsed_data: dict) -> List[ValidationError]:
    """
    Validate CFDI business rules.

    Rules:
    1. Total > 0
    2. Total = SubTotal - Descuento + Impuestos
    3. Receptor.Rfc != Emisor.Rfc
    4. If IVA exists: rate in [0%, 8%, 16%]
    """
    errors = []

    # Rule 1: Total > 0
    if parsed_data.get('total', 0) <= 0:
        errors.append(ValidationError(
            field="total",
            message="Total must be greater than 0",
            severity=ValidationSeverity.ERROR,
            rule="TOTAL_POSITIVE"
        ))

    # Rule 2: Total calculation
    subtotal = parsed_data.get('subtotal', 0)
    descuento = parsed_data.get('descuento', 0)
    impuestos = parsed_data.get('impuestos_total', 0)
    total = parsed_data.get('total', 0)

    expected_total = subtotal - descuento + impuestos
    if abs(total - expected_total) > Decimal('0.01'):  # Allow 1 cent rounding
        errors.append(ValidationError(
            field="total",
            message=f"Total calculation mismatch: Total={total}, expected SubTotal({subtotal}) - Descuento({descuento}) + Impuestos({impuestos}) = {expected_total}",
            severity=ValidationSeverity.ERROR,
            expected=str(expected_total),
            actual=str(total),
            rule="TOTAL_CALCULATION"
        ))

    # Rule 3: No self-invoicing
    receptor_rfc = parsed_data.get('rfc')
    emisor_rfc = parsed_data.get('emisor_rfc')
    if receptor_rfc and emisor_rfc and receptor_rfc == emisor_rfc:
        errors.append(ValidationError(
            field="rfc",
            message="Receptor RFC cannot be same as Emisor RFC (no self-invoicing)",
            severity=ValidationSeverity.ERROR,
            rule="NO_SELF_INVOICE"
        ))

    # Rule 4: Valid IVA rates
    iva_rate = parsed_data.get('iva_rate')
    if iva_rate is not None:
        valid_rates = [Decimal('0.00'), Decimal('0.08'), Decimal('0.16')]
        if iva_rate not in valid_rates:
            errors.append(ValidationError(
                field="iva_rate",
                message=f"Invalid IVA rate: {iva_rate}",
                severity=ValidationSeverity.WARNING,
                expected="0%, 8%, or 16%",
                actual=f"{iva_rate * 100}%",
                rule="IVA_RATE_INVALID"
            ))

    return errors
```

---

## Usage Examples

### Example 1: Basic XML Validation

```python
validator = FacturaValidator()

# Read XML file
with open("factura.xml", "rb") as f:
    xml_content = f.read()

# Validate
result = validator.validate_xml(xml_content)

if result.is_valid:
    print("✓ Factura is valid")
    print(f"  Version: CFDI {result.version}")
else:
    print("✗ Validation failed:")
    for error in result.get_errors():
        print(f"  [{error.severity.value}] {error.field}: {error.message}")
```

### Example 2: Validate Individual Fields

```python
# Validate RFC
rfc_result = validator.validate_rfc("ABC123456XYZ")
if rfc_result.is_valid:
    print("✓ RFC is valid")
else:
    print(f"✗ RFC invalid: {rfc_result.errors[0].message}")

# Validate UUID
uuid_result = validator.validate_uuid("12345678-1234-4234-a234-123456789012")
if uuid_result.is_valid:
    print("✓ UUID is valid")

# Validate amount
amount_result = validator.validate_amount(Decimal("500.00"))
if amount_result.is_valid:
    print("✓ Amount is valid")

# Validate date
date_result = validator.validate_date(date(2024, 11, 1))
if date_result.is_valid:
    print("✓ Date is valid")
```

### Example 3: Full Validation with Signature and SAT Database

```python
validator = FacturaValidator(sat_api_client=sat_client)

result = validator.validate_xml(
    xml_content=xml_content,
    validate_signature=True,      # Verify digital signature
    verify_sat_database=True      # Check UUID in SAT database
)

if result.is_valid:
    print("✓ Fully validated (schema + signature + SAT database)")
else:
    print("✗ Validation failed:")
    for error in result.errors:
        print(f"  [{error.severity.value}] {error.field}: {error.message}")
```

### Example 4: Handle Warnings

```python
result = validator.validate_xml(xml_content)

if result.has_errors():
    print("✗ Validation failed with errors:")
    for error in result.get_errors():
        print(f"  {error.field}: {error.message}")
else:
    print("✓ No errors")

    if result.get_warnings():
        print("⚠ Warnings:")
        for warning in result.get_warnings():
            print(f"  {warning.field}: {warning.message}")
```

---

## Multi-Domain Examples

### Healthcare: HL7 Message Validator

```python
class HL7Validator:
    """
    Validate HL7 v2.x messages (Healthcare).

    Similar pattern:
    - Parse HL7 segments (MSH, PID, OBR, etc.)
    - Validate field types (string, numeric, date)
    - Check value sets (coding systems)
    - Verify checksums
    """

    def validate_message(self, hl7_content: str) -> ValidationResult:
        # Parse HL7 segments
        # Validate MSH segment (header)
        # Validate PID segment (patient)
        # Validate business rules
        pass
```

### Legal: PACER XML Validator

```python
class PACERValidator:
    """
    Validate PACER court filing XML (Legal).

    Similar pattern:
    - Parse PACER XML
    - Validate case number format
    - Check filing date constraints
    - Verify document type codes
    """

    def validate_filing(self, xml_content: bytes) -> ValidationResult:
        # Parse PACER XML
        # Validate case_number format
        # Check filing_date
        # Verify document_type
        pass
```

### Research: NSF FastLane XML Validator

```python
class FastLaneValidator:
    """
    Validate NSF grant submission XML (Research).

    Similar pattern:
    - Parse grant XML
    - Validate grant ID format
    - Check budget calculations
    - Verify institution credentials
    """

    def validate_submission(self, xml_content: bytes) -> ValidationResult:
        # Parse grant XML
        # Validate grant_id
        # Check budget totals
        # Verify institution
        pass
```

---

## Validation Rules Reference

### RFC Validation Rules

| Rule ID | Check | Severity | Example |
|---------|-------|----------|---------|
| RFC_LENGTH | Length 12 or 13 | ERROR | ✓ "ABC123456XYZ" (12) |
| RFC_PATTERN | Alphanumeric pattern | ERROR | ✗ "ABC@123456XYZ" |
| RFC_GENERIC | Not test/generic | WARNING | ⚠ "XAXX010101000" |

### UUID Validation Rules

| Rule ID | Check | Severity | Example |
|---------|-------|----------|---------|
| UUID_FORMAT | 8-4-4-4-12 hex | ERROR | ✓ "12345678-1234-4234-a234-123456789012" |
| UUID_VERSION | Version 4 | WARNING | ⚠ UUIDv1 (should be v4) |

### Amount Validation Rules

| Rule ID | Check | Severity | Example |
|---------|-------|----------|---------|
| AMOUNT_POSITIVE | > 0 | ERROR | ✗ -500.00, ✗ 0.00 |
| AMOUNT_DECIMALS | Max 2 decimals | ERROR | ✗ 500.123 |
| AMOUNT_MAX | < 1 billion | WARNING | ⚠ 1,500,000,000 |

### Date Validation Rules

| Rule ID | Check | Severity | Example |
|---------|-------|----------|---------|
| DATE_FUTURE | Not in future | ERROR | ✗ 2030-01-01 (if today is 2025) |
| DATE_TOO_OLD | >= 2000-01-01 | WARNING | ⚠ 1999-12-31 |

### Business Rules

| Rule ID | Check | Severity | Example |
|---------|-------|----------|---------|
| TOTAL_POSITIVE | Total > 0 | ERROR | ✗ Total = -100 |
| TOTAL_CALCULATION | Total = SubTotal - Descuento + Impuestos | ERROR | ✗ Total mismatch |
| NO_SELF_INVOICE | Receptor ≠ Emisor | ERROR | ✗ Same RFC |
| IVA_RATE_INVALID | Rate in [0%, 8%, 16%] | WARNING | ⚠ 12% IVA |

---

## Error Handling

### Custom Exceptions

```python
class ValidatorError(Exception):
    """Base exception for validator errors."""
    pass

class SATAPIError(ValidatorError):
    """SAT API unavailable or error."""
    def __init__(self, message: str):
        self.message = message
        super().__init__(message)

class SchemaValidationError(ValidatorError):
    """XSD schema validation failed."""
    pass

class SignatureValidationError(ValidatorError):
    """Digital signature invalid."""
    pass
```

---

## Performance Characteristics

### Validation Latencies

| Operation | Expected Latency (p95) | Notes |
|-----------|------------------------|-------|
| `validate_rfc()` | <5ms | Regex matching |
| `validate_uuid()` | <5ms | Format check |
| `validate_amount()` | <1ms | Numeric checks |
| `validate_date()` | <1ms | Date comparison |
| `validate_xml()` (basic) | <100ms | Parse + validate |
| `validate_xml()` (+ signature) | <300ms | Includes crypto |
| `validate_xml()` (+ SAT API) | <2000ms | External API call |

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: RFC validation
def test_validate_rfc_valid_person()
def test_validate_rfc_valid_company()
def test_validate_rfc_invalid_length()
def test_validate_rfc_invalid_pattern()
def test_validate_rfc_generic()

# Test: UUID validation
def test_validate_uuid_valid()
def test_validate_uuid_invalid_format()
def test_validate_uuid_wrong_version()

# Test: Amount validation
def test_validate_amount_valid()
def test_validate_amount_negative()
def test_validate_amount_zero()
def test_validate_amount_too_many_decimals()

# Test: Date validation
def test_validate_date_valid()
def test_validate_date_future()
def test_validate_date_too_old()

# Test: Full XML validation
def test_validate_xml_cfdi_33()
def test_validate_xml_cfdi_40()
def test_validate_xml_invalid_version()
def test_validate_xml_malformed()
def test_validate_xml_business_rules()
```

---

## Related Primitives

- **FacturaStore** (OL) - Uses FacturaValidator for validation before storage
- **StorageEngine** (from Vertical 1.1) - Stores validated XML files
- **FacturaUploadDialog** (IL) - Shows validation errors to user

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 2-3 days
**Dependencies:** XML parser (lxml or ElementTree), XSD schema files (CFDI 3.3/4.0), Optional: cryptography (signature validation), SAT API client (UUID verification)
