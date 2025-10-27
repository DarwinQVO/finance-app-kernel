# Parsing Logic (Bank of America Specific)

> **Purpose:** Define how to extract transactions from Bank of America PDF statements

---

## Bank of America Statement Format

**What we're parsing:** Monthly checking/savings account statements in PDF format

**Typical file characteristics:**
- **File size:** 100 KB - 2 MB
- **Pages:** 2-4 pages typically
- **Format:** Digital PDF (not scanned - has selectable text)
- **Encoding:** UTF-8

---

## PDF Structure

**Page 1: Account Summary**
```
BANK OF AMERICA
Checking Account Statement

Account Number: ****1234
Statement Period: October 1-31, 2024

Beginning Balance (10/01): $2,450.32
Ending Balance (10/31):    $1,873.19

Deposits/Credits:   $4,200.00
Withdrawals/Debits: $4,777.13
```

**Page 2+: Transaction Details**
```
Date        Description                           Amount     Balance
10/01/2024  Beginning Balance                                $2,450.32
10/02/2024  PAYCHECK DEPOSIT                     $2,100.00   $4,550.32
10/03/2024  WHOLE FOODS MARKET #1234 SAN FR      -$87.43     $4,462.89
10/04/2024  NETFLIX.COM                          -$14.99     $4,447.90
10/05/2024  SHELL GAS #5678 OAKLAND CA           -$45.00     $4,402.90
...
10/31/2024  Ending Balance                                   $1,873.19
```

---

## Parsing Strategy

### Step 1: Identify Transaction Table

**Look for table header:**
```
Date        Description                           Amount     Balance
```

**Characteristics:**
- Usually starts on page 2
- Header row has 4 columns: Date, Description, Amount, Balance
- May have slight variations: "Transaction Date" vs "Date"

**PyPDF2 approach:**
```python
import PyPDF2

def find_transaction_table(pdf_path):
    with open(pdf_path, 'rb') as file:
        reader = PyPDF2.PdfReader(file)

        for page_num, page in enumerate(reader.pages):
            text = page.extract_text()

            # Look for table header
            if 'Date' in text and 'Description' in text and 'Amount' in text:
                return page_num, text

    raise ValueError("Transaction table not found in PDF")
```

### Step 2: Extract Transaction Rows

**Pattern matching:**

Each transaction line follows this format:
```
MM/DD/YYYY  DESCRIPTION (variable length)  $XXX.XX  $XXX.XX
```

**Regex pattern:**
```python
import re

TRANSACTION_PATTERN = re.compile(
    r'(\d{2}/\d{2}/\d{4})\s+'     # Date: 10/15/2024
    r'(.+?)\s+'                    # Description: anything (non-greedy)
    r'(-?\$[\d,]+\.\d{2})\s+'     # Amount: -$87.43 or $2,100.00
    r'(\$[\d,]+\.\d{2})'          # Balance: $4,462.89
)

def extract_transactions(text):
    transactions = []

    for match in TRANSACTION_PATTERN.finditer(text):
        date_raw = match.group(1)       # "10/15/2024"
        desc_raw = match.group(2).strip()  # "WHOLE FOODS MARKET #1234"
        amount_raw = match.group(3)     # "-$87.43"
        balance_raw = match.group(4)    # "$4,462.89"

        transactions.append({
            'date_raw': date_raw,
            'description_raw': desc_raw,
            'amount_raw': amount_raw,
            'balance_raw': balance_raw
        })

    return transactions
```

### Step 3: Clean and Validate

**Date cleaning:**
```python
from datetime import datetime

def parse_date(date_raw):
    """Convert '10/15/2024' to '2024-10-15' (ISO 8601)"""
    try:
        dt = datetime.strptime(date_raw, '%m/%d/%Y')
        return dt.strftime('%Y-%m-%d')
    except ValueError:
        raise ValueError(f"Invalid date format: {date_raw}")
```

**Amount parsing:**
```python
def parse_amount(amount_raw):
    """Convert '-$87.43' to -87.43 (float)"""
    # Remove $ and commas
    amount_str = amount_raw.replace('$', '').replace(',', '')
    return float(amount_str)
```

**Description cleaning:**
```python
def clean_description(desc_raw):
    """Remove excessive whitespace, keep original text"""
    return ' '.join(desc_raw.split())
```

---

## Edge Cases

### 1. Pending Transactions

**What they look like:**
```
10/31/2024  PENDING: UBER TRIP #ABC123         -$18.50*   $1,854.69
```

**Handling:**
- Asterisk (*) indicates pending
- Include in observations with flag: `pending=true`
- User can filter pending vs. settled transactions

```python
def is_pending(desc_raw, amount_raw):
    """Check if transaction is pending"""
    return (
        'PENDING:' in desc_raw.upper() or
        amount_raw.endswith('*')
    )
```

### 2. Refunds/Credits

**What they look like:**
```
10/15/2024  REFUND: AMAZON.COM ORDER #123      $45.99     $4,508.88
```

**Handling:**
- Positive amount = credit/refund
- Keep as separate transaction (don't try to match with original purchase)
- User can manually link refund to original transaction if desired

### 3. Multi-Line Descriptions

**What they look like:**
```
10/15/2024  TRANSFER FROM SAVINGS
            ACCOUNT ****5678                   $500.00    $5,008.88
```

**Handling:**
- Description spans multiple lines
- Parse as single transaction with concatenated description
- Remove extra whitespace

```python
def handle_multiline(text):
    """Combine multi-line descriptions"""
    lines = text.split('\n')

    current_tx = None
    transactions = []

    for line in lines:
        match = TRANSACTION_PATTERN.match(line)
        if match:
            # New transaction starts
            if current_tx:
                transactions.append(current_tx)
            current_tx = extract_transaction(match)
        elif current_tx:
            # Continuation of previous description
            current_tx['description_raw'] += ' ' + line.strip()

    if current_tx:
        transactions.append(current_tx)

    return transactions
```

### 4. Fees and Charges

**What they look like:**
```
10/31/2024  MONTHLY SERVICE FEE                -$12.00    $1,861.19
10/31/2024  OVERDRAFT FEE                      -$35.00    $1,826.19
```

**Handling:**
- Parse like regular transactions
- Auto-categorize as "Fees" or "Bank Charges"
- User can dispute fees (note field)

### 5. Interest Earned

**What they look like:**
```
10/31/2024  INTEREST EARNED THIS PERIOD        $0.12      $1,873.31
```

**Handling:**
- Small positive amount (< $1 typically)
- Auto-categorize as "Interest Income"
- Include in income totals

### 6. Beginning/Ending Balance Rows

**What they look like:**
```
10/01/2024  Beginning Balance                             $2,450.32
10/31/2024  Ending Balance                                $1,873.19
```

**Handling:**
- **Skip these rows** - they're not transactions
- Use for validation (last transaction balance should match ending balance)

```python
def should_skip_row(desc_raw):
    """Skip non-transaction rows"""
    skip_keywords = [
        'BEGINNING BALANCE',
        'ENDING BALANCE',
        'SUBTOTAL',
        'TOTAL DEPOSITS',
        'TOTAL WITHDRAWALS'
    ]
    return any(kw in desc_raw.upper() for kw in skip_keywords)
```

### 7. Foreign Currency Transactions

**What they look like:**
```
10/15/2024  RESTAURANT PARIS EUR 45.00
            EXCHANGE RATE 1.10                -$49.50    $4,413.39
10/15/2024  FOREIGN TRANSACTION FEE            -$2.50    $4,410.89
```

**Handling:**
- Two separate transactions:
  1. Primary charge (EUR 45.00 converted to $49.50)
  2. FX fee ($2.50)
- Parse both, link later in relationship detection (see user journey #12)
- Extract foreign currency amount from description if present

```python
def extract_fx_info(desc_raw):
    """Extract foreign currency info from description"""
    # Pattern: "EUR 45.00" or "EXCHANGE RATE 1.10"
    currency_match = re.search(r'([A-Z]{3})\s+([\d.]+)', desc_raw)
    rate_match = re.search(r'EXCHANGE RATE\s+([\d.]+)', desc_raw)

    return {
        'foreign_currency': currency_match.group(1) if currency_match else None,
        'foreign_amount': float(currency_match.group(2)) if currency_match else None,
        'exchange_rate': float(rate_match.group(1)) if rate_match else None
    }
```

### 8. Truncated Descriptions

**What they look like:**
```
10/15/2024  AMAZON MKTPLACE PMTS AMZN.COM/BI...  -$45.99   $4,418.90
```

**Handling:**
- BoFA truncates long descriptions with "..."
- Store as-is (raw observation)
- Normalize merchant name during counterparty resolution
- Original untruncated description is lost (not in PDF)

### 9. Check Transactions

**What they look like:**
```
10/15/2024  CHECK #1234                        -$150.00   $4,268.90
```

**Handling:**
- Extract check number from description
- Store as transaction with type: "check"
- User can add payee manually in notes field

### 10. ATM Withdrawals

**What they look like:**
```
10/15/2024  ATM WITHDRAWAL
            7-ELEVEN #5678 SAN FRANCISCO CA    -$40.00    $4,228.90
10/15/2024  ATM FEE                            -$2.50     $4,226.40
```

**Handling:**
- Two transactions: withdrawal + fee
- Parse both separately
- Link via relationship detection (same date, "ATM FEE" in description)

---

## Validation Rules

### Transaction-Level Validation

**Required fields:**
```python
def validate_transaction(tx):
    errors = []

    # Date must be valid
    if not is_valid_date(tx['date_raw']):
        errors.append(f"Invalid date: {tx['date_raw']}")

    # Amount must be non-zero
    if parse_amount(tx['amount_raw']) == 0:
        errors.append("Amount cannot be zero")

    # Description cannot be empty
    if not tx['description_raw'].strip():
        errors.append("Description cannot be empty")

    return errors
```

### Statement-Level Validation

**Balance reconciliation:**
```python
def validate_statement(transactions, beginning_balance, ending_balance):
    """Verify transactions add up to ending balance"""

    calculated_balance = beginning_balance

    for tx in transactions:
        calculated_balance += parse_amount(tx['amount_raw'])

    difference = abs(calculated_balance - ending_balance)

    if difference > 0.01:  # Allow 1 cent rounding error
        raise ValueError(
            f"Balance mismatch: expected {ending_balance}, "
            f"calculated {calculated_balance} (diff: ${difference:.2f})"
        )
```

**Transaction count check:**
```python
def validate_count(transactions, min_expected=0, max_expected=1000):
    """Sanity check on transaction count"""
    count = len(transactions)

    if count < min_expected:
        raise ValueError(f"Too few transactions: {count} (expected ≥{min_expected})")

    if count > max_expected:
        raise ValueError(f"Too many transactions: {count} (expected ≤{max_expected})")
```

---

## Parser Implementation

**Complete parser function:**

```python
def parse_bofa_statement(pdf_path):
    """
    Extract transactions from Bank of America PDF statement.

    Returns:
        {
            'upload_id': 'UL_abc123',
            'transactions': [
                {
                    'row_id': 0,
                    'date_raw': '10/15/2024',
                    'description_raw': 'WHOLE FOODS MARKET #1234',
                    'amount_raw': '-$87.43',
                    'balance_raw': '$4,462.89',
                    'pending': False
                },
                ...
            ],
            'metadata': {
                'account_number': '****1234',
                'statement_period': 'October 1-31, 2024',
                'beginning_balance': 2450.32,
                'ending_balance': 1873.19,
                'page_count': 3
            }
        }
    """

    # Step 1: Open PDF
    with open(pdf_path, 'rb') as file:
        reader = PyPDF2.PdfReader(file)

        # Step 2: Extract text from all pages
        full_text = ''
        for page in reader.pages:
            full_text += page.extract_text() + '\n'

        # Step 3: Extract metadata (account number, dates, balances)
        metadata = extract_metadata(full_text)

        # Step 4: Find transaction table
        table_start = full_text.find('Date')  # Find table header
        if table_start == -1:
            raise ValueError("Transaction table not found")

        table_text = full_text[table_start:]

        # Step 5: Extract transactions
        raw_transactions = extract_transactions(table_text)

        # Step 6: Filter out non-transaction rows
        transactions = [
            tx for tx in raw_transactions
            if not should_skip_row(tx['description_raw'])
        ]

        # Step 7: Add row IDs
        for idx, tx in enumerate(transactions):
            tx['row_id'] = idx

        # Step 8: Validate
        validate_count(transactions, min_expected=1)
        validate_statement(
            transactions,
            metadata['beginning_balance'],
            metadata['ending_balance']
        )

        return {
            'upload_id': generate_upload_id(),
            'transactions': transactions,
            'metadata': metadata
        }
```

---

## Performance Characteristics

**Typical statement (2 pages, 42 transactions):**
- Parse time: 0.5 - 2 seconds
- Memory usage: < 10 MB
- No external API calls

**Large statement (4 pages, 200 transactions):**
- Parse time: 2 - 5 seconds
- Memory usage: < 20 MB

**Edge case: Corrupted PDF:**
- PyPDF2 raises exception
- Return user-friendly error: "Could not read PDF - file may be corrupted"

---

## Error Handling

**Error types and user-facing messages:**

| Error | User Message | Suggested Action |
|-------|-------------|------------------|
| PDF file corrupted | "Could not read PDF - file may be corrupted" | "Try downloading the statement again from your bank" |
| Transaction table not found | "This doesn't look like a Bank of America statement" | "Make sure you uploaded a BoFA checking/savings statement" |
| Balance mismatch | "Statement balance doesn't match transactions" | "This might be a partial statement - contact support" |
| No transactions found | "No transactions found in this statement" | "This statement period may have no activity" |
| Date parsing error | "Found invalid date in statement" | "Contact support with the file name" |

---

## Testing Strategy

**Golden file approach:**

```
tests/fixtures/
  bofa-statement-typical.pdf       → 42 transactions, expected output
  bofa-statement-pending.pdf       → Contains pending transactions
  bofa-statement-refund.pdf        → Contains refunds
  bofa-statement-fx.pdf            → Contains foreign currency
  bofa-statement-zero-txns.pdf     → No transactions (valid)
  bofa-statement-corrupted.pdf     → Corrupted PDF (should fail gracefully)
```

**Test cases:**

```python
def test_parse_typical_statement():
    result = parse_bofa_statement('tests/fixtures/bofa-statement-typical.pdf')

    assert len(result['transactions']) == 42
    assert result['metadata']['beginning_balance'] == 2450.32
    assert result['metadata']['ending_balance'] == 1873.19

def test_parse_pending_transactions():
    result = parse_bofa_statement('tests/fixtures/bofa-statement-pending.pdf')

    pending_txns = [tx for tx in result['transactions'] if tx.get('pending')]
    assert len(pending_txns) > 0

def test_parse_zero_transactions():
    result = parse_bofa_statement('tests/fixtures/bofa-statement-zero-txns.pdf')

    assert len(result['transactions']) == 0
    # Should succeed, not error

def test_parse_corrupted_pdf():
    with pytest.raises(ValueError, match="Could not read PDF"):
        parse_bofa_statement('tests/fixtures/bofa-statement-corrupted.pdf')
```

---

## Future Enhancements

**When to add support for other statement types:**

1. **Credit Card Statements** (different format)
   - Separate parser: `parse_bofa_credit_card()`
   - Different table structure (has category column)

2. **Investment Accounts** (very different)
   - Trades, dividends, capital gains
   - Requires new parser entirely

3. **Other Banks** (Chase, Wells Fargo)
   - Each bank has unique PDF format
   - Parsers registered in ParserRegistry
   - User selects bank during upload or system auto-detects

**Auto-detection (future):**
```python
def detect_statement_type(pdf_path):
    """Automatically detect bank and account type"""
    text = extract_first_page_text(pdf_path)

    if 'BANK OF AMERICA' in text:
        if 'CHECKING' in text or 'SAVINGS' in text:
            return 'bofa_checking'
        elif 'CREDIT CARD' in text:
            return 'bofa_credit_card'

    elif 'CHASE' in text:
        return 'chase_checking'

    # ... more banks

    return 'unknown'
```

---

## Summary

**This parser is:**
- ✅ Bank of America specific (checking/savings)
- ✅ Handles common edge cases (pending, refunds, FX)
- ✅ Validates output (balance reconciliation)
- ✅ Fast (< 5 seconds for typical statement)

**This parser does NOT:**
- ❌ Handle scanned/image PDFs (requires OCR)
- ❌ Support other banks (yet)
- ❌ Parse credit card statements (different format)
- ❌ Extract check images (not in PDF)

**Key Principle:**
Parse conservatively - when in doubt, include the transaction and let normalization handle edge cases. It's better to have extra data than missing data.
