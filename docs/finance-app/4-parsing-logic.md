# Lógica de Parsing (Específico de Bank of America)

> **Propósito:** Definir cómo extraer transacciones de estados de cuenta PDF de Bank of America

---

## Formato de Estado de Cuenta de Bank of America

**Lo que estamos parseando:** Estados de cuenta mensuales de cuentas corrientes/ahorro en formato PDF

**Características típicas del archivo:**
- **Tamaño del archivo:** 100 KB - 2 MB
- **Páginas:** 2-4 páginas típicamente
- **Formato:** PDF digital (no escaneado - tiene texto seleccionable)
- **Codificación:** UTF-8

---

## Estructura del PDF

**Página 1: Resumen de Cuenta**
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

**Página 2+: Detalles de Transacciones**
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

## Estrategia de Parsing

### Paso 1: Identificar Tabla de Transacciones

**Buscar encabezado de tabla:**
```
Date        Description                           Amount     Balance
```

**Características:**
- Usualmente comienza en la página 2
- La fila de encabezado tiene 4 columnas: Date, Description, Amount, Balance
- Puede tener ligeras variaciones: "Transaction Date" vs "Date"

**Enfoque con PyPDF2:**
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

### Paso 2: Extraer Filas de Transacciones

**Coincidencia de patrones:**

Cada línea de transacción sigue este formato:
```
MM/DD/YYYY  DESCRIPTION (variable length)  $XXX.XX  $XXX.XX
```

**Patrón regex:**
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

### Paso 3: Limpiar y Validar

**Limpieza de fechas:**
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

**Parsing de montos:**
```python
def parse_amount(amount_raw):
    """Convert '-$87.43' to -87.43 (float)"""
    # Remove $ and commas
    amount_str = amount_raw.replace('$', '').replace(',', '')
    return float(amount_str)
```

**Limpieza de descripción:**
```python
def clean_description(desc_raw):
    """Remove excessive whitespace, keep original text"""
    return ' '.join(desc_raw.split())
```

---

## Casos Especiales

### 1. Transacciones Pendientes

**Cómo se ven:**
```
10/31/2024  PENDING: UBER TRIP #ABC123         -$18.50*   $1,854.69
```

**Manejo:**
- El asterisco (*) indica pendiente
- Incluir en observaciones con bandera: `pending=true`
- El usuario puede filtrar transacciones pendientes vs. liquidadas

```python
def is_pending(desc_raw, amount_raw):
    """Check if transaction is pending"""
    return (
        'PENDING:' in desc_raw.upper() or
        amount_raw.endswith('*')
    )
```

### 2. Reembolsos/Créditos

**Cómo se ven:**
```
10/15/2024  REFUND: AMAZON.COM ORDER #123      $45.99     $4,508.88
```

**Manejo:**
- Monto positivo = crédito/reembolso
- Mantener como transacción separada (no intentar emparejar con compra original)
- El usuario puede vincular manualmente el reembolso a la transacción original si lo desea

### 3. Descripciones Multi-Línea

**Cómo se ven:**
```
10/15/2024  TRANSFER FROM SAVINGS
            ACCOUNT ****5678                   $500.00    $5,008.88
```

**Manejo:**
- La descripción abarca múltiples líneas
- Parsear como transacción única con descripción concatenada
- Remover espacios en blanco extra

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

### 4. Cargos y Comisiones

**Cómo se ven:**
```
10/31/2024  MONTHLY SERVICE FEE                -$12.00    $1,861.19
10/31/2024  OVERDRAFT FEE                      -$35.00    $1,826.19
```

**Manejo:**
- Parsear como transacciones regulares
- Auto-categorizar como "Cargos" o "Comisiones Bancarias"
- El usuario puede disputar cargos (campo de notas)

### 5. Intereses Ganados

**Cómo se ven:**
```
10/31/2024  INTEREST EARNED THIS PERIOD        $0.12      $1,873.31
```

**Manejo:**
- Monto positivo pequeño (< $1 típicamente)
- Auto-categorizar como "Ingresos por Intereses"
- Incluir en totales de ingresos

### 6. Filas de Saldo Inicial/Final

**Cómo se ven:**
```
10/01/2024  Beginning Balance                             $2,450.32
10/31/2024  Ending Balance                                $1,873.19
```

**Manejo:**
- **Omitir estas filas** - no son transacciones
- Usar para validación (el saldo de la última transacción debe coincidir con el saldo final)

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

### 7. Transacciones en Moneda Extranjera

**Cómo se ven:**
```
10/15/2024  RESTAURANT PARIS EUR 45.00
            EXCHANGE RATE 1.10                -$49.50    $4,413.39
10/15/2024  FOREIGN TRANSACTION FEE            -$2.50    $4,410.89
```

**Manejo:**
- Dos transacciones separadas:
  1. Cargo principal (EUR 45.00 convertido a $49.50)
  2. Comisión FX ($2.50)
- Parsear ambas, vincular más tarde en detección de relaciones (ver user journey #12)
- Extraer monto en moneda extranjera de la descripción si está presente

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

### 8. Descripciones Truncadas

**Cómo se ven:**
```
10/15/2024  AMAZON MKTPLACE PMTS AMZN.COM/BI...  -$45.99   $4,418.90
```

**Manejo:**
- BoFA trunca descripciones largas con "..."
- Almacenar tal cual (observación cruda)
- Normalizar nombre del comerciante durante resolución de contraparte
- La descripción original no truncada se pierde (no está en el PDF)

### 9. Transacciones de Cheques

**Cómo se ven:**
```
10/15/2024  CHECK #1234                        -$150.00   $4,268.90
```

**Manejo:**
- Extraer número de cheque de la descripción
- Almacenar como transacción con tipo: "cheque"
- El usuario puede agregar beneficiario manualmente en campo de notas

### 10. Retiros de Cajero Automático

**Cómo se ven:**
```
10/15/2024  ATM WITHDRAWAL
            7-ELEVEN #5678 SAN FRANCISCO CA    -$40.00    $4,228.90
10/15/2024  ATM FEE                            -$2.50     $4,226.40
```

**Manejo:**
- Dos transacciones: retiro + comisión
- Parsear ambas por separado
- Vincular via detección de relaciones (misma fecha, "ATM FEE" en descripción)

---

## Reglas de Validación

### Validación a Nivel de Transacción

**Campos requeridos:**
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

### Validación a Nivel de Estado de Cuenta

**Reconciliación de saldo:**
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

**Verificación de conteo de transacciones:**
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

## Implementación del Parser

**Función completa del parser:**

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

## Características de Rendimiento

**Estado de cuenta típico (2 páginas, 42 transacciones):**
- Tiempo de parsing: 0.5 - 2 segundos
- Uso de memoria: < 10 MB
- Sin llamadas a APIs externas

**Estado de cuenta grande (4 páginas, 200 transacciones):**
- Tiempo de parsing: 2 - 5 segundos
- Uso de memoria: < 20 MB

**Caso especial: PDF corrupto:**
- PyPDF2 lanza excepción
- Retornar error amigable: "No se pudo leer el PDF - el archivo puede estar corrupto"

---

## Manejo de Errores

**Tipos de error y mensajes para el usuario:**

| Error | Mensaje al Usuario | Acción Sugerida |
|-------|-------------------|-----------------|
| PDF file corrupted | "No se pudo leer el PDF - el archivo puede estar corrupto" | "Intenta descargar el estado de cuenta nuevamente desde tu banco" |
| Transaction table not found | "Esto no parece un estado de cuenta de Bank of America" | "Asegúrate de haber subido un estado de cuenta corriente/ahorro de BoFA" |
| Balance mismatch | "El saldo del estado de cuenta no coincide con las transacciones" | "Esto podría ser un estado de cuenta parcial - contacta soporte" |
| No transactions found | "No se encontraron transacciones en este estado de cuenta" | "Este período de estado de cuenta puede no tener actividad" |
| Date parsing error | "Se encontró fecha inválida en el estado de cuenta" | "Contacta soporte con el nombre del archivo" |

---

## Estrategia de Testing

**Enfoque de archivos golden:**

```
tests/fixtures/
  bofa-statement-typical.pdf       → 42 transactions, expected output
  bofa-statement-pending.pdf       → Contains pending transactions
  bofa-statement-refund.pdf        → Contains refunds
  bofa-statement-fx.pdf            → Contains foreign currency
  bofa-statement-zero-txns.pdf     → No transactions (valid)
  bofa-statement-corrupted.pdf     → Corrupted PDF (should fail gracefully)
```

**Casos de prueba:**

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

## Mejoras Futuras

**Cuándo agregar soporte para otros tipos de estados de cuenta:**

1. **Estados de Cuenta de Tarjeta de Crédito** (formato diferente)
   - Parser separado: `parse_bofa_credit_card()`
   - Estructura de tabla diferente (tiene columna de categoría)

2. **Cuentas de Inversión** (muy diferente)
   - Operaciones, dividendos, ganancias de capital
   - Requiere un parser completamente nuevo

3. **Otros Bancos** (Chase, Wells Fargo)
   - Cada banco tiene un formato PDF único
   - Parsers registrados en ParserRegistry
   - El usuario selecciona banco durante subida o el sistema auto-detecta

**Auto-detección (futuro):**
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

## Resumen

**Este parser es:**
- ✅ Específico de Bank of America (corriente/ahorro)
- ✅ Maneja casos especiales comunes (pendientes, reembolsos, FX)
- ✅ Valida salida (reconciliación de saldo)
- ✅ Rápido (< 5 segundos para estado de cuenta típico)

**Este parser NO:**
- ❌ Maneja PDFs escaneados/imagen (requiere OCR)
- ❌ Soporta otros bancos (aún)
- ❌ Parsea estados de cuenta de tarjeta de crédito (formato diferente)
- ❌ Extrae imágenes de cheques (no están en el PDF)

**Principio Clave:**
Parsear conservadoramente - en caso de duda, incluir la transacción y dejar que la normalización maneje casos especiales. Es mejor tener datos extra que datos faltantes.
