# AccountValidator (OL Primitive)

**Vertical:** 3.1 Account Registry
**Layer:** Objective Layer (OL)
**Type:** Validation Engine
**Status:** ✅ Specified

---

## Purpose

Validates account data (names, types, currencies, uniqueness) before persistence to AccountStore.

**Core Responsibility:** Ensure data quality and enforce business rules for account entities.

---

## Multi-Domain Applicability

Validation patterns apply to ANY closed registry:

| Domain | Entity | Validations |
|--------|--------|-------------|
| **Finance** | Account | unique name, valid type (checking/credit), ISO 4217 currency |
| **Healthcare** | Insurance Provider | unique name, valid type (HMO/PPO), valid policy number format |
| **Legal** | Trust Account | unique name, valid jurisdiction code, valid account type (trust/IOLTA) |
| **Research** | Funding Source | unique name, valid funding type, positive amount, future expiration |
| **Manufacturing** | Cost Center | unique code, valid department type, positive budget |
| **Media** | Revenue Stream | unique name, valid platform, valid payout method |

---

## Interface Contract

### Python Interface

```python
from typing import List, Optional
from dataclasses import dataclass

@dataclass
class ValidationResult:
    is_valid: bool
    errors: List[ValidationError]

@dataclass
class ValidationError:
    field: str
    code: str
    message: str
    details: Optional[dict] = None

class AccountValidator:
    """
    Validates account data before persistence.

    Enforces:
    - Name format and length
    - Type in allowed list
    - Currency is ISO 4217 code
    - Uniqueness (when combined with AccountStore)
    """

    ALLOWED_TYPES = ["checking", "savings", "credit", "debit", "investment", "loan"]

    # ISO 4217 currency codes (subset - can be extended)
    ALLOWED_CURRENCIES = [
        "USD", "MXN", "EUR", "GBP", "CAD", "JPY", "CNY", "INR", "BRL", "AUD",
        "CHF", "SEK", "NOK", "DKK", "PLN", "CZK", "HUF", "RON", "RUB", "TRY",
        "ZAR", "NZD", "SGD", "HKD", "KRW", "THB", "MYR", "IDR", "PHP", "AED"
    ]

    def __init__(self):
        pass

    def validate_create(
        self,
        name: str,
        type: str,
        currency: str,
        institution: Optional[str] = None
    ) -> ValidationResult:
        """
        Validate data for account creation.

        Args:
            name: Account name
            type: Account type
            currency: ISO 4217 currency code
            institution: Optional institution name

        Returns:
            ValidationResult with is_valid and errors list
        """
        errors = []

        # Validate each field
        name_error = self.validate_name(name)
        if name_error:
            errors.append(name_error)

        type_error = self.validate_type(type)
        if type_error:
            errors.append(type_error)

        currency_error = self.validate_currency(currency)
        if currency_error:
            errors.append(currency_error)

        if institution:
            institution_error = self.validate_institution(institution)
            if institution_error:
                errors.append(institution_error)

        return ValidationResult(
            is_valid=len(errors) == 0,
            errors=errors
        )

    def validate_update(
        self,
        updates: dict,
        immutable_fields: List[str] = ["currency", "type"]
    ) -> ValidationResult:
        """
        Validate data for account update.

        Args:
            updates: Dictionary of fields to update
            immutable_fields: Fields that cannot be updated

        Returns:
            ValidationResult with is_valid and errors list
        """
        errors = []

        # Check for immutable fields
        for field in immutable_fields:
            if field in updates:
                errors.append(ValidationError(
                    field=field,
                    code="IMMUTABLE_FIELD",
                    message=f"Field '{field}' is immutable and cannot be updated",
                    details={"field": field}
                ))

        # Validate mutable fields
        if "name" in updates:
            name_error = self.validate_name(updates["name"])
            if name_error:
                errors.append(name_error)

        if "institution" in updates:
            institution_error = self.validate_institution(updates["institution"])
            if institution_error:
                errors.append(institution_error)

        return ValidationResult(
            is_valid=len(errors) == 0,
            errors=errors
        )

    def validate_name(self, name: str) -> Optional[ValidationError]:
        """
        Validate account name.

        Rules:
        - Required (length >= 1)
        - Max length 100 characters
        - Allowed characters: letters, numbers, spaces, hyphens, apostrophes
        - Pattern: ^[A-Za-z0-9\s\-']+$

        Returns:
            ValidationError if invalid, None if valid
        """
        if not name or len(name.strip()) == 0:
            return ValidationError(
                field="name",
                code="NAME_REQUIRED",
                message="Account name is required"
            )

        if len(name) > 100:
            return ValidationError(
                field="name",
                code="NAME_TOO_LONG",
                message=f"Account name too long ({len(name)} chars). Maximum 100 characters.",
                details={"length": len(name), "max_length": 100}
            )

        # Check allowed characters
        import re
        if not re.match(r"^[A-Za-z0-9\s\-']+$", name):
            return ValidationError(
                field="name",
                code="NAME_INVALID_CHARACTERS",
                message="Account name contains invalid characters. Only letters, numbers, spaces, hyphens, and apostrophes allowed.",
                details={"pattern": "^[A-Za-z0-9\\s\\-']+$"}
            )

        return None

    def validate_type(self, type: str) -> Optional[ValidationError]:
        """
        Validate account type.

        Rules:
        - Required
        - Must be in ALLOWED_TYPES

        Returns:
            ValidationError if invalid, None if valid
        """
        if not type:
            return ValidationError(
                field="type",
                code="TYPE_REQUIRED",
                message="Account type is required"
            )

        if type not in self.ALLOWED_TYPES:
            return ValidationError(
                field="type",
                code="INVALID_TYPE",
                message=f"Invalid account type '{type}'. Must be one of: {', '.join(self.ALLOWED_TYPES)}",
                details={"allowed_values": self.ALLOWED_TYPES}
            )

        return None

    def validate_currency(self, currency: str) -> Optional[ValidationError]:
        """
        Validate currency code.

        Rules:
        - Required
        - Must be ISO 4217 3-letter code
        - Must be uppercase
        - Must be in ALLOWED_CURRENCIES

        Returns:
            ValidationError if invalid, None if valid
        """
        if not currency:
            return ValidationError(
                field="currency",
                code="CURRENCY_REQUIRED",
                message="Currency is required"
            )

        if len(currency) != 3:
            return ValidationError(
                field="currency",
                code="INVALID_CURRENCY_FORMAT",
                message=f"Currency must be 3-letter ISO 4217 code (e.g., USD, MXN). Got: '{currency}'",
                details={"expected_length": 3, "actual_length": len(currency)}
            )

        if currency != currency.upper():
            return ValidationError(
                field="currency",
                code="CURRENCY_NOT_UPPERCASE",
                message=f"Currency code must be uppercase. Got: '{currency}', expected: '{currency.upper()}'",
                details={"value": currency, "expected": currency.upper()}
            )

        if currency not in self.ALLOWED_CURRENCIES:
            return ValidationError(
                field="currency",
                code="UNSUPPORTED_CURRENCY",
                message=f"Unsupported currency '{currency}'. Supported: {', '.join(self.ALLOWED_CURRENCIES[:10])}...",
                details={"allowed_currencies": self.ALLOWED_CURRENCIES}
            )

        return None

    def validate_institution(self, institution: str) -> Optional[ValidationError]:
        """
        Validate institution name (optional field).

        Rules:
        - Max length 200 characters
        - No validation on content (free text)

        Returns:
            ValidationError if invalid, None if valid
        """
        if len(institution) > 200:
            return ValidationError(
                field="institution",
                code="INSTITUTION_TOO_LONG",
                message=f"Institution name too long ({len(institution)} chars). Maximum 200 characters.",
                details={"length": len(institution), "max_length": 200}
            )

        return None
```

---

## Usage Examples

### Example 1: Validate Account Creation (Success)

```python
validator = AccountValidator()

result = validator.validate_create(
    name="BofA Checking",
    type="checking",
    currency="USD",
    institution="Bank of America"
)

print(result.is_valid)  # True
print(result.errors)    # []
```

### Example 2: Validate Account Creation (Multiple Errors)

```python
validator = AccountValidator()

result = validator.validate_create(
    name="",  # Empty name
    type="invalid_type",  # Not in ALLOWED_TYPES
    currency="usd",  # Lowercase (should be uppercase)
    institution="A" * 250  # Too long
)

print(result.is_valid)  # False
print(len(result.errors))  # 4

for error in result.errors:
    print(f"{error.field}: {error.message}")

# Output:
# name: Account name is required
# type: Invalid account type 'invalid_type'. Must be one of: checking, savings, credit, debit, investment, loan
# currency: Currency code must be uppercase. Got: 'usd', expected: 'USD'
# institution: Institution name too long (250 chars). Maximum 200 characters.
```

### Example 3: Validate Account Update (Immutable Field Rejected)

```python
validator = AccountValidator()

result = validator.validate_update(
    updates={
        "name": "BofA Personal Checking",
        "currency": "MXN"  # Trying to change currency (immutable)
    }
)

print(result.is_valid)  # False
print(result.errors[0].code)  # "IMMUTABLE_FIELD"
print(result.errors[0].message)  # "Field 'currency' is immutable and cannot be updated"
```

### Example 4: Validate Name with Invalid Characters

```python
validator = AccountValidator()

name_error = validator.validate_name("BofA <script>alert('xss')</script>")

print(name_error.code)  # "NAME_INVALID_CHARACTERS"
print(name_error.message)  # "Account name contains invalid characters..."
```

### Example 5: Integration with AccountStore

```python
# In AccountStore.create()
def create(self, user_id, name, type, currency, institution=None):
    # 1. Validate input
    validator = AccountValidator()
    validation_result = validator.validate_create(name, type, currency, institution)

    if not validation_result.is_valid:
        # Convert validation errors to exception
        error = validation_result.errors[0]
        if error.code == "INVALID_TYPE":
            raise InvalidAccountTypeError(error.message, error.details["allowed_values"])
        elif error.code == "UNSUPPORTED_CURRENCY":
            raise InvalidCurrencyError(error.message, currency)
        else:
            raise ValidationError(error.message)

    # 2. Check uniqueness (separate from validator)
    self._check_duplicate_name(user_id, name)

    # 3. Persist to database
    account_id = self._generate_account_id(name)
    # ... INSERT INTO accounts ...

    return Account(...)
```

---

## Multi-Domain Examples

### Healthcare: Insurance Provider Validator

```python
class InsuranceProviderValidator:
    ALLOWED_TYPES = ["HMO", "PPO", "EPO", "Medicare", "Medicaid"]

    def validate_policy_number(self, policy_number: str) -> Optional[ValidationError]:
        """
        Validate insurance policy number.

        Rules:
        - Required
        - Alphanumeric only
        - Length 8-20 characters
        """
        if not policy_number:
            return ValidationError(
                field="policy_number",
                code="POLICY_NUMBER_REQUIRED",
                message="Policy number is required"
            )

        if not policy_number.isalnum():
            return ValidationError(
                field="policy_number",
                code="INVALID_POLICY_NUMBER_FORMAT",
                message="Policy number must be alphanumeric only"
            )

        if len(policy_number) < 8 or len(policy_number) > 20:
            return ValidationError(
                field="policy_number",
                code="INVALID_POLICY_NUMBER_LENGTH",
                message=f"Policy number must be 8-20 characters. Got: {len(policy_number)}"
            )

        return None

# Usage:
validator = InsuranceProviderValidator()
result = validator.validate_create(
    name="Blue Cross Blue Shield",
    type="PPO",
    policy_number="BC12345678",
    network="National"
)
```

### Legal: Trust Account Validator

```python
class TrustAccountValidator:
    ALLOWED_TYPES = ["trust", "operating", "iolta"]
    ALLOWED_JURISDICTIONS = ["CA", "NY", "TX", "FL", "IL"]  # US state codes

    def validate_jurisdiction(self, jurisdiction: str) -> Optional[ValidationError]:
        """
        Validate jurisdiction code.

        Rules:
        - Required
        - Must be 2-letter US state code
        - Must be in ALLOWED_JURISDICTIONS
        """
        if not jurisdiction:
            return ValidationError(
                field="jurisdiction",
                code="JURISDICTION_REQUIRED",
                message="Jurisdiction is required"
            )

        if jurisdiction not in self.ALLOWED_JURISDICTIONS:
            return ValidationError(
                field="jurisdiction",
                code="INVALID_JURISDICTION",
                message=f"Invalid jurisdiction '{jurisdiction}'. Must be one of: {', '.join(self.ALLOWED_JURISDICTIONS)}"
            )

        return None

# Usage:
validator = TrustAccountValidator()
result = validator.validate_create(
    name="IOLTA Trust Account",
    type="iolta",
    jurisdiction="CA",
    account_number="TR987654321"
)
```

---

## Error Codes Reference

| Code | Field | Description | Example |
|------|-------|-------------|---------|
| `NAME_REQUIRED` | name | Name is empty or whitespace | `name=""` |
| `NAME_TOO_LONG` | name | Name exceeds 100 characters | `name="A"*101` |
| `NAME_INVALID_CHARACTERS` | name | Name contains disallowed characters | `name="BofA<script>"` |
| `TYPE_REQUIRED` | type | Type is missing | `type=None` |
| `INVALID_TYPE` | type | Type not in ALLOWED_TYPES | `type="invalid"` |
| `CURRENCY_REQUIRED` | currency | Currency is missing | `currency=None` |
| `INVALID_CURRENCY_FORMAT` | currency | Currency not 3 letters | `currency="US"` |
| `CURRENCY_NOT_UPPERCASE` | currency | Currency is lowercase | `currency="usd"` |
| `UNSUPPORTED_CURRENCY` | currency | Currency not in ALLOWED_CURRENCIES | `currency="XYZ"` |
| `INSTITUTION_TOO_LONG` | institution | Institution name exceeds 200 characters | `institution="A"*201` |
| `IMMUTABLE_FIELD` | (various) | Trying to update immutable field | `updates={"currency": "MXN"}` |

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Name validation
def test_validate_name_success()
def test_validate_name_required()
def test_validate_name_too_long()
def test_validate_name_invalid_characters()

# Test: Type validation
def test_validate_type_success()
def test_validate_type_required()
def test_validate_type_invalid()

# Test: Currency validation
def test_validate_currency_success()
def test_validate_currency_required()
def test_validate_currency_wrong_length()
def test_validate_currency_not_uppercase()
def test_validate_currency_unsupported()

# Test: Institution validation
def test_validate_institution_success()
def test_validate_institution_too_long()

# Test: Create validation (combined)
def test_validate_create_all_valid()
def test_validate_create_multiple_errors()

# Test: Update validation
def test_validate_update_success()
def test_validate_update_immutable_field()
def test_validate_update_invalid_name()
```

---

## Performance Characteristics

**Validation Performance:**
- Name validation: <1ms (regex match)
- Type validation: <1ms (set membership check)
- Currency validation: <1ms (set membership check)
- Full validation: <5ms (all fields combined)

**Note:** Validation is synchronous and in-memory (no database queries).

---

## Related Primitives

- **AccountStore** (OL) - Uses AccountValidator before persistence
- **ValidationEngine** (from Vertical 1.3) - Similar validation pattern for normalization
- **AccountManager** (IL) - Uses AccountValidator for client-side validation (same rules)

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 1 day
**Dependencies:** None (pure validation logic)
