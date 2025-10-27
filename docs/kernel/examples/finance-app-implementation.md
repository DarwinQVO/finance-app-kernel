# Finance App Implementation: Simple App Uses Complex Kernel

**Purpose:** This document shows how Darwin's personal finance app (simple requirements: 500 tx/month, SQLite, PyPDF2) uses the universal kernel primitives (designed for 10M+ scale).

**Key Insight:** The same kernel interfaces work at ANY scale. Personal use cherry-picks 20% of features; Enterprise uses 100%.

---

## Document Structure

This mapping follows the 12 user journeys from `/docs/finance-app/1-core-user-journeys.md` and shows:
1. **Journey narrative** (what Darwin does)
2. **Kernel primitives used** (which abstractions)
3. **Configuration** (Personal profile: what's enabled/disabled)
4. **What's NOT used** (enterprise features ignored)

---

## Journey 1: When User Uploads Document

### User Story
> Darwin opens the finance app on Sunday morning, clicks "Upload Statement", selects `BoFA_Oct2024.pdf` (2.3MB), uploads it. System accepts immediately without questions.

### Kernel Primitives Used

#### 1. HashCalculator (Personal Profile)
```python
# Calculate file hash for deduplication
hash_calculator = HashCalculator(algorithm="sha256")
file_hash = hash_calculator.calculate_hash(pdf_bytes)  # In-memory (file < 10MB)
# Returns: "sha256:abc123..."
```

**Features Used:**
- ✅ `calculateHash()` - In-memory hashing (2.3MB fits in RAM)
- ✅ SHA-256 algorithm

**Features NOT Used:**
- ❌ `calculateHashStreaming()` - File < 10MB (no streaming needed)
- ❌ `verifyHash()` on retrieval - Trust filesystem
- ❌ Multi-algorithm support - SHA-256 sufficient
- ❌ Malware hash checking - No threat intel database

**Configuration:**
```yaml
hash_calculator:
  algorithm: "sha256"
  chunk_size_kb: 8  # Not used (in-memory mode)
  verify_on_retrieve: false
```

---

#### 2. StorageEngine (Personal Profile)
```python
# Store PDF with automatic deduplication
storage_engine = FilesystemStorageEngine(base_path="~/.finance-app/storage")
storage_ref = storage_engine.store(pdf_bytes, metadata={"mime_type": "application/pdf"})
# Returns: "sha256:abc123..." (same as hash, used as storage key)
```

**Features Used:**
- ✅ `store()` - Write file to filesystem
- ✅ `exists()` - Check if hash already stored (dedupe)
- ✅ Content-addressable storage - Hash = storage key
- ✅ Filesystem backend - `~/.finance-app/storage/content/ab/c1/23...`

**Features NOT Used:**
- ❌ S3 backend - Local files only
- ❌ Streaming upload - File < 50MB (in-memory)
- ❌ Compression - PDFs already compressed
- ❌ Encryption at rest - Local machine trusted
- ❌ GC policy - Keep files forever (no deletion)
- ❌ Metrics/logging - No Prometheus
- ❌ Quota enforcement - Disk space not constrained

**Configuration:**
```yaml
storage_engine:
  backend: "filesystem"
  base_path: "/Users/darwin/.finance-app/storage"
  hash_algorithm: "sha256"
  max_file_size_mb: 50
  compression: false
  gc_policy:
    enabled: false
```

---

#### 3. FileArtifact (Personal Profile)
```python
# Create metadata record linking to storage
artifact = FileArtifact.create(
    file_hash=file_hash,
    storage_ref=storage_ref,
    file_name="BoFA_Oct2024.pdf",
    file_size_bytes=2300000,
    mime_type="application/pdf",
    source_type="bofa_pdf",
    uploaded_at=datetime.now()
)
# Returns: FileArtifact(file_id="FILE_001", ...)
```

**Features Used:**
- ✅ Core metadata - `file_name`, `file_size_bytes`, `mime_type`, `file_hash`
- ✅ Upload context - `uploaded_at`, `source_type`
- ✅ Dedupe by hash - Query: `SELECT * FROM file_artifacts WHERE file_hash = ?`
- ✅ Storage reference - Link to StorageEngine

**Features NOT Used:**
- ❌ `ref_count` - No GC (never delete files)
- ❌ `last_accessed_at` - No access tracking
- ❌ Virus scanning - Trust local files
- ❌ Thumbnail generation - PDFs don't need previews
- ❌ OCR text extraction - No full-text search
- ❌ Extended metadata (tags, categories)

**Database Schema (SQLite):**
```sql
CREATE TABLE file_artifacts (
    file_id TEXT PRIMARY KEY,
    file_hash TEXT UNIQUE NOT NULL,
    storage_ref TEXT NOT NULL,
    file_name TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    mime_type TEXT NOT NULL,
    source_type TEXT NOT NULL,
    uploaded_at DATETIME NOT NULL
);
```

---

#### 4. UploadRecord (Personal Profile)
```python
# Create upload record (state machine entry point)
upload_record = UploadRecord.create(
    upload_id=generate_upload_id(),  # "UL_abc123"
    file_id=artifact.file_id,
    status="queued_for_parse",
    upload_log_ref="/logs/UL_abc123.upload.log"
)
# Returns: UploadRecord(upload_id="UL_abc123", status="queued_for_parse")
```

**Features Used:**
- ✅ Core state machine - Track status (queued → parsing → parsed → normalized)
- ✅ Progressive disclosure - `parse_log_ref` appears after parsing
- ✅ Error tracking - `error_message` if parsing fails
- ✅ Single-user FIFO - Process in order

**Features NOT Used:**
- ❌ Automatic retries - User manually re-uploads if error
- ❌ Timeout monitoring - Parser fast (<5s)
- ❌ Priority queue - Single user
- ❌ `retry_count`, `max_retries` fields
- ❌ `parsing_started_at` timeout fields
- ❌ Webhooks - User polls status manually
- ❌ Metrics - No Prometheus

**Database Schema (SQLite):**
```sql
CREATE TABLE upload_records (
    upload_id TEXT PRIMARY KEY,
    file_id TEXT NOT NULL REFERENCES file_artifacts(file_id),
    status TEXT NOT NULL CHECK (status IN ('queued_for_parse', 'parsing', 'parsed', 'normalizing', 'normalized', 'error')),
    error_message TEXT,
    upload_log_ref TEXT NOT NULL,
    parse_log_ref TEXT,
    normalizer_log_ref TEXT,
    observations_count INTEGER,
    canonicals_count INTEGER,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

### What's NOT Used (Enterprise Features)

From these 4 primitives, Darwin's app ignores:
- **StorageEngine:** S3, streaming, encryption, GC, metrics, quotas
- **HashCalculator:** Streaming, verification on retrieval, malware detection
- **FileArtifact:** Virus scanning, thumbnails, OCR, PII detection, audit logs
- **UploadRecord:** Retries, timeouts, priority queues, webhooks, Prometheus

**Lines of Code:**
- Personal implementation: ~300 lines total
- Enterprise implementation: ~4200 lines total
- **Darwin uses 7% of the code that Enterprise needs**

---

## Journey 2: When System Does Parsing

### User Story
> After upload, system immediately starts parsing (no queue delay for single user). Parser extracts 42 transactions from BoFA PDF in 3 seconds.

### Kernel Primitives Used

#### 1. Parser (Personal Profile - Hardcoded)
```python
# Single hardcoded parser (only BoFA supported)
class BoFAPDFParser(Parser):
    parser_id = "bofa_pdf_parser"
    version = "1.0.0"
    supported_source_types = ["bofa_pdf"]

    def parse(self, content: bytes) -> List[Observation]:
        # Extract text from PDF
        pdf = PyPDF2.PdfReader(io.BytesIO(content))
        text = "".join(page.extract_text() for page in pdf.pages)

        # Find transaction table
        pattern = r'(\d{2}/\d{2}/\d{4})\s+(.+?)\s+(-?\$[\d,]+\.\d{2})\s+(\$[\d,]+\.\d{2})'
        matches = re.findall(pattern, text)

        # Extract observations (AS-IS, no validation)
        observations = []
        for row_id, (date, desc, amount, balance) in enumerate(matches):
            observations.append(Observation(
                raw_data={
                    "date": date,              # "10/15/2024" (string, AS-IS)
                    "description": desc,       # "WHOLE FOODS MARKET" (AS-IS)
                    "amount": amount,          # "-$87.43" (string, AS-IS)
                    "balance": balance         # "$4,462.89" (string, AS-IS)
                },
                row_id=row_id
            ))

        return observations
```

**Features Used:**
- ✅ `parse()` method - Extract transactions
- ✅ AS-IS extraction - Strings, no validation
- ✅ Deterministic row IDs - Same file → same row_id
- ✅ Zero rows = success - Empty statement valid
- ✅ Basic error handling - `ParseError` if PDF corrupted

**Features NOT Used:**
- ❌ ParserRegistry - Only 1 parser (hardcoded)
- ❌ `can_parse()` detection - Only BoFA PDFs uploaded
- ❌ Versioning - No parser version tracking
- ❌ Warnings vs errors - All issues are errors (simpler)
- ❌ Partial parse - Either all rows or fail
- ❌ Performance metrics

**Implementation:**
- Single file: `bofa_pdf_parser.py` (~200 lines)
- Dependency: PyPDF2 (stdlib)
- No tests (manual verification)

---

#### 2. ObservationStore (Personal Profile)
```python
# Store raw observations (AS-IS from parser)
observation_store = ObservationStore(db_path="~/.finance-app/data.db")

for observation in parser_result.observations:
    observation_store.upsert(
        upload_id="UL_abc123",
        row_id=observation.row_id,
        raw_data=observation.raw_data
    )
# Stores 42 observations with idempotent upsert
```

**Features Used:**
- ✅ Idempotent upsert - `(upload_id, row_id)` = unique key
- ✅ Raw storage - JSON blob, no validation
- ✅ Basic queries - `SELECT * FROM observations WHERE upload_id = ?`

**Features NOT Used:**
- ❌ Bitemporal tracking - No `valid_from`/`valid_to` fields
- ❌ Version history - No tracking of updates
- ❌ Indexes on raw_data fields - No JSON querying
- ❌ Partitioning - Single SQLite file
- ❌ Compression - Data small enough

**Database Schema (SQLite):**
```sql
CREATE TABLE observations (
    upload_id TEXT NOT NULL,
    row_id INTEGER NOT NULL,
    raw_data TEXT NOT NULL,  -- JSON blob
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (upload_id, row_id)
);
```

---

#### 3. ParseLog (Personal Profile)
```python
# Log parse execution (simple, no structured logging)
parse_log = {
    "upload_id": "UL_abc123",
    "parser_id": "bofa_pdf_parser",
    "started_at": "2024-10-27T10:00:00Z",
    "completed_at": "2024-10-27T10:00:03Z",
    "duration_seconds": 3,
    "observations_extracted": 42,
    "errors": [],
    "warnings": []
}
# Written to: /logs/UL_abc123.parse.json
```

**Features Used:**
- ✅ Basic execution log - Start/end time, duration
- ✅ Observation count
- ✅ Error tracking

**Features NOT Used:**
- ❌ Structured logging (JSON to centralized system)
- ❌ Trace IDs (distributed tracing)
- ❌ Parser version tracking
- ❌ Performance metrics (p95 latency)
- ❌ Warning categorization

**Storage:** Simple JSON file, not indexed

---

#### 4. UploadRecord State Transition (Coordinator)
```python
# Coordinator updates status (inline, not background job)
coordinator = Coordinator()

# Before parsing
coordinator.transition(upload_record, "parsing")
# upload_record.status = "parsing"

# After parsing success
coordinator.transition(
    upload_record,
    "parsed",
    observations_count=42,
    parse_log_ref="/logs/UL_abc123.parse.json"
)
# upload_record.status = "parsed"
# upload_record.observations_count = 42
# upload_record.parse_log_ref = "/logs/UL_abc123.parse.json"
```

**Features Used:**
- ✅ Status transitions - queued → parsing → parsed
- ✅ Progressive disclosure - `parse_log_ref` added when parsing completes

**Features NOT Used:**
- ❌ Background coordinator job - Runs inline (single user)
- ❌ Event bus - Direct function calls
- ❌ Status transition webhooks

---

### Execution Flow (Personal Profile)

```
1. Upload completes → UploadRecord created (status="queued_for_parse")
2. API immediately calls parser (no queue delay, single user)
3. Coordinator: transition("parsing")
4. BoFAPDFParser.parse(pdf_bytes) → 42 observations (3 seconds)
5. ObservationStore.upsert() × 42
6. ParseLog written to /logs/UL_abc123.parse.json
7. Coordinator: transition("parsed", observations_count=42, parse_log_ref="...")
8. User polls /uploads/UL_abc123 → sees status="parsed"
```

**Total latency:** ~3 seconds (parsing dominates)

---

### What's NOT Used (Enterprise Features)

- **Parser:** Registry, versioning, canary deploys, A/B testing, auto-rollback
- **ObservationStore:** Bitemporal, partitioning, compression, JSON indexes
- **ParseLog:** Structured logging, Prometheus, trace IDs
- **Coordinator:** Background jobs, event bus, webhooks

**Lines of Code:**
- Personal: ~350 lines (parser + store + log)
- Enterprise: ~6000 lines
- **Darwin uses 6% of Enterprise code**

---

## Journey 3: When User Navigates OL (Objective Layer)

### User Story
> Darwin clicks "View Transactions" tab. System shows list of 42 raw observations (AS-IS from parser, no normalization yet). He sees messy data: "WHOLE FOODS MARKET" vs "WHOLE FOODS #1234", dates as strings "10/15/2024".

### Kernel Primitives Used

#### 1. TransactionQuery (Personal Profile - Simplified)
```python
# Simple query: Get all observations for an upload
def get_observations(upload_id: str) -> List[Observation]:
    query = """
        SELECT upload_id, row_id, raw_data, created_at
        FROM observations
        WHERE upload_id = ?
        ORDER BY row_id
    """
    rows = db.execute(query, [upload_id]).fetchall()

    return [
        Observation(
            raw_data=json.loads(row["raw_data"]),
            row_id=row["row_id"]
        )
        for row in rows
    ]
```

**Features Used:**
- ✅ Basic SELECT query
- ✅ Filter by `upload_id`
- ✅ Order by `row_id` (chronological as appeared in PDF)

**Features NOT Used:**
- ❌ Complex filters (date ranges, amount ranges)
- ❌ Full-text search
- ❌ Pagination - 42 rows fit in single page
- ❌ Sorting by fields - Default order (row_id) is fine
- ❌ Aggregations (SUM, COUNT by category)
- ❌ Joins with CanonicalStore
- ❌ Query optimization (indexes, caching)

**Implementation:** ~30 lines Python

---

#### 2. UI Component: ObservationListView (Personal Profile)
```typescript
// Simple table rendering raw observations
function ObservationListView({ observations }) {
  return (
    <table>
      <thead>
        <tr>
          <th>Row</th>
          <th>Date</th>
          <th>Description</th>
          <th>Amount</th>
          <th>Balance</th>
        </tr>
      </thead>
      <tbody>
        {observations.map(obs => (
          <tr key={obs.row_id}>
            <td>{obs.row_id}</td>
            <td>{obs.raw_data.date}</td>
            <td>{obs.raw_data.description}</td>
            <td>{obs.raw_data.amount}</td>
            <td>{obs.raw_data.balance}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**Features Used:**
- ✅ Simple HTML table
- ✅ Display AS-IS (no formatting)
- ✅ Row ID for debugging

**Features NOT Used:**
- ❌ Virtual scrolling - 42 rows (small dataset)
- ❌ Column sorting
- ❌ Inline editing
- ❌ Row selection / bulk actions
- ❌ Export to CSV
- ❌ Responsive mobile view (desktop-only for v1)

**Implementation:** ~50 lines React

---

### What Darwin Sees (OL = Raw Data)

```
Row | Date       | Description              | Amount    | Balance
----+------------+--------------------------+-----------+----------
0   | 10/01/2024 | BEGINNING BALANCE        | $0.00     | $5,200.00
1   | 10/03/2024 | PAYCHECK DEPOSIT         | $2,100.00 | $7,300.00
2   | 10/05/2024 | WHOLE FOODS MARKET       | -$87.43   | $7,212.57
3   | 10/07/2024 | WHOLE FOODS #1234        | -$45.20   | $7,167.37
...
41  | 10/30/2024 | ENDING BALANCE           | $0.00     | $4,462.89
```

**Note:** Data is AS-IS (messy):
- "WHOLE FOODS MARKET" ≠ "WHOLE FOODS #1234" (not normalized yet)
- Dates are strings, not parsed
- Amounts have "$" and "," (not numeric)

---

### What's NOT Used (Enterprise Features)

- **TransactionQuery:** Pagination, indexes, caching, complex filters, aggregations
- **UI:** Virtual scrolling, sorting, editing, mobile responsive, export

**Lines of Code:**
- Personal: ~80 lines (query + UI)
- Enterprise: ~800 lines
- **Darwin uses 10% of Enterprise code**

---

## Journey 4: When User Navigates RL (Representation Layer)

### User Story
> Darwin clicks "Normalized View" tab. System shows 42 cleaned transactions: "Whole Foods Market" (normalized), dates as Date objects, amounts as numbers (-87.43). Categories assigned (Groceries), duplicates merged.

### Kernel Primitives Used

#### 1. Normalizer (Personal Profile - Simple Rules)
```python
# Simple normalizer: Clean strings, parse dates/amounts
class SimpleNormalizer(Normalizer):
    def normalize(self, observations: List[Observation]) -> List[Canonical]:
        canonicals = []

        for obs in observations:
            # Parse date (assume MM/DD/YYYY)
            date_str = obs.raw_data["date"]
            date = datetime.strptime(date_str, "%m/%d/%Y")

            # Parse amount (remove $ and ,)
            amount_str = obs.raw_data["amount"].replace("$", "").replace(",", "")
            amount = float(amount_str)

            # Clean merchant name
            merchant = self.clean_merchant(obs.raw_data["description"])

            # Create canonical transaction
            canonical = CanonicalTransaction(
                date=date,
                amount=amount,
                merchant=merchant,
                category=None,  # Auto-categorize next
                account="Bank of America Checking"
            )

            canonicals.append(canonical)

        return canonicals

    def clean_merchant(self, raw: str) -> str:
        # Simple rules
        raw = raw.strip().upper()

        # Remove trailing numbers (e.g., "WHOLE FOODS #1234" → "WHOLE FOODS")
        raw = re.sub(r'\s+#\d+$', '', raw)

        # Known mappings
        mappings = {
            "WHOLE FOODS MARKET": "Whole Foods Market",
            "WHOLE FOODS": "Whole Foods Market",
            "STARBUCKS": "Starbucks",
            # ... 20 more hardcoded rules
        }

        return mappings.get(raw, raw.title())
```

**Features Used:**
- ✅ Basic normalization - Parse dates, amounts
- ✅ Merchant name cleaning - Simple regex + mappings
- ✅ Hardcoded rules - 20-30 common merchants

**Features NOT Used:**
- ❌ Machine learning - No ML model for merchant normalization
- ❌ Fuzzy matching - No Levenshtein distance
- ❌ External APIs - No Google Places for merchant data
- ❌ User feedback loop - No correction learning
- ❌ Multi-currency support
- ❌ NormalizationLog with versions
- ❌ Undo/redo normalization

**Implementation:** ~150 lines Python

---

#### 2. CanonicalStore (Personal Profile)
```python
# Store normalized transactions
canonical_store = CanonicalStore(db_path="~/.finance-app/data.db")

for canonical in normalizer_result.canonicals:
    canonical_store.upsert(
        upload_id="UL_abc123",
        row_id=canonical.row_id,
        canonical_data={
            "date": canonical.date.isoformat(),
            "amount": canonical.amount,
            "merchant": canonical.merchant,
            "category": canonical.category,
            "account": canonical.account
        }
    )
```

**Features Used:**
- ✅ Idempotent upsert - `(upload_id, row_id)` = key
- ✅ Store normalized data

**Features NOT Used:**
- ❌ Bitemporal tracking - No version history
- ❌ Conflict resolution - No merging strategy
- ❌ Indexes on canonical fields
- ❌ Partitioning
- ❌ Fuzzy deduplication

**Database Schema (SQLite):**
```sql
CREATE TABLE canonicals (
    upload_id TEXT NOT NULL,
    row_id INTEGER NOT NULL,
    date DATE NOT NULL,
    amount REAL NOT NULL,
    merchant TEXT NOT NULL,
    category TEXT,
    account TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (upload_id, row_id)
);

CREATE INDEX idx_date ON canonicals(date);
CREATE INDEX idx_merchant ON canonicals(merchant);
```

---

#### 3. TransactionQuery (RL View)
```python
# Query normalized transactions
def get_canonicals(upload_id: str) -> List[CanonicalTransaction]:
    query = """
        SELECT date, amount, merchant, category, account
        FROM canonicals
        WHERE upload_id = ?
        ORDER BY date DESC
    """
    rows = db.execute(query, [upload_id]).fetchall()

    return [CanonicalTransaction(**row) for row in rows]
```

**Features Used:**
- ✅ SELECT from canonicals table
- ✅ Order by date (chronological)

---

#### 4. UI Component: CanonicalListView
```typescript
// Render normalized transactions
function CanonicalListView({ transactions }) {
  return (
    <table>
      <thead>
        <tr>
          <th>Date</th>
          <th>Merchant</th>
          <th>Amount</th>
          <th>Category</th>
          <th>Account</th>
        </tr>
      </thead>
      <tbody>
        {transactions.map(tx => (
          <tr key={tx.id}>
            <td>{formatDate(tx.date)}</td>
            <td>{tx.merchant}</td>
            <td className={tx.amount < 0 ? 'expense' : 'income'}>
              ${Math.abs(tx.amount).toFixed(2)}
            </td>
            <td>{tx.category || '—'}</td>
            <td>{tx.account}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**Features Used:**
- ✅ Simple table with formatting
- ✅ Color-coding (expense = red, income = green)
- ✅ Date formatting

---

### What Darwin Sees (RL = Cleaned Data)

```
Date       | Merchant           | Amount   | Category  | Account
-----------+--------------------+----------+-----------+-------------------
2024-10-30 | Ending Balance     | $0.00    | —         | BoFA Checking
2024-10-28 | Starbucks          | $5.75    | Dining    | BoFA Checking
2024-10-07 | Whole Foods Market | $45.20   | Groceries | BoFA Checking
2024-10-05 | Whole Foods Market | $87.43   | Groceries | BoFA Checking
2024-10-03 | Paycheck Deposit   | $2,100.00| Income    | BoFA Checking
```

**Note:** Data is CLEANED:
- "WHOLE FOODS #1234" → "Whole Foods Market" (normalized)
- Dates are Date objects (sortable, filterable)
- Amounts are numbers (math operations possible)

---

### What's NOT Used (Enterprise Features)

- **Normalizer:** ML models, fuzzy matching, external APIs, feedback loops, multi-currency
- **CanonicalStore:** Bitemporal, conflict resolution, partitioning
- **UI:** Advanced filtering, editing, export, mobile

**Lines of Code:**
- Personal: ~280 lines (normalizer + store + query + UI)
- Enterprise: ~3500 lines
- **Darwin uses 8% of Enterprise code**

---

## Summary: What % of Kernel Darwin Uses

| Primitive | Personal LOC | Enterprise LOC | % Used |
|-----------|--------------|----------------|--------|
| StorageEngine | 150 | 1500 | 10% |
| HashCalculator | 30 | 800 | 4% |
| UploadRecord | 100 | 1200 | 8% |
| FileArtifact | 50 | 1000 | 5% |
| ArtifactRetriever | 80 | 900 | 9% |
| Parser | 200 | 5000 | 4% |
| ObservationStore | 100 | 1200 | 8% |
| ParseLog | 50 | 400 | 13% |
| Normalizer | 150 | 2500 | 6% |
| CanonicalStore | 100 | 1400 | 7% |
| TransactionQuery | 60 | 600 | 10% |
| UI Components | 150 | 1500 | 10% |
| **TOTAL** | **1,220** | **18,000** | **7%** |

**Key Finding:** Darwin's personal finance app uses **~7% of the code** that an Enterprise implementation would need.

The same interfaces work at both scales. Darwin hardcodes, skips tests, ignores metrics. Enterprise does the opposite. Same primitives, different configuration.

---

## Journey 5: When System Detects Duplicate (Apple Card)

### User Story
> Darwin uploads Apple Card statement. System detects transaction "APPLE.COM/BILL $9.99" appears in both BoFA (debit from checking) and Apple Card (charge to card). System shows duplicate warning: "This transaction may already exist in BoFA Checking."

### Kernel Primitives Used

#### 1. FuzzyMatcher (Personal Profile - SKIPPED for v1)
```python
# Darwin's v1: NO duplicate detection
# Accepts all transactions as unique (simplicity over accuracy)
# User manually identifies duplicates (rare edge case)
```

**Decision:** **Skip this primitive entirely for Personal v1**

**Rationale:**
- Duplicate detection requires fuzzy matching (date ±2 days, amount exact, merchant similar)
- Complex logic: Levenshtein distance, threshold tuning
- Rare edge case for Darwin (Apple Pay duplicates happen <5 times/year)
- Manual correction easier than building auto-detection

**What Enterprise Would Use:**
- ✅ FuzzyMatcher primitive - Levenshtein distance for merchant names
- ✅ Time window matching - Date ±2 days
- ✅ Amount exact match - Penny-perfect
- ✅ Cluster confidence scores - 0.0-1.0 probability
- ✅ User review UI - "Merge these 2 transactions?"

**Implementation Complexity:**
- Personal: 0 lines (feature not built)
- Enterprise: ~1500 lines (fuzzy matching algorithm + UI)
- **Darwin skips 100% of this feature**

---

### Alternative: Manual Duplicate Handling

```python
# Darwin's approach: Manual identification
# User sees duplicate in UI, clicks "Mark as duplicate" button
# System hides one transaction from view (soft delete)

def mark_as_duplicate(transaction_id: str, duplicate_of: str):
    db.execute("""
        UPDATE canonicals
        SET is_duplicate = TRUE, duplicate_of = ?
        WHERE id = ?
    """, [duplicate_of, transaction_id])

# Query excludes duplicates
query = "SELECT * FROM canonicals WHERE is_duplicate = FALSE"
```

**Features Used:**
- ✅ Boolean flag `is_duplicate`
- ✅ Manual user action

**Features NOT Used:**
- ❌ Automatic detection
- ❌ Fuzzy matching
- ❌ Confidence scores
- ❌ Cluster algorithms

**Lines of Code:**
- Manual approach: ~20 lines
- Auto-detection: ~1500 lines
- **Manual is 99% simpler**

---

## Journey 6: When System Does Account Resolution

### User Story
> Darwin uploads statements from 2 accounts: "BoFA Checking", "Apple Card". System needs to link transactions to correct account.

### Kernel Primitives Used

#### 1. AccountStore (Personal Profile - Hardcoded)
```python
# Hardcoded accounts (Darwin only has 2)
ACCOUNTS = {
    "bofa_checking": Account(
        id="acc_bofa_checking",
        name="Bank of America Checking",
        type="checking"
    ),
    "apple_card": Account(
        id="acc_apple_card",
        name="Apple Card",
        type="credit_card"
    )
}

# Link upload to account (user selects at upload time)
def create_upload(file, account_name):
    upload_record = UploadRecord.create(
        file_id=file.id,
        account_id=ACCOUNTS[account_name].id
    )
```

**Features Used:**
- ✅ Hardcoded accounts dictionary
- ✅ User selects account at upload (dropdown: "BoFA Checking" or "Apple Card")
- ✅ Foreign key: `canonicals.account_id → accounts.id`

**Features NOT Used:**
- ❌ AccountStore table - Hardcoded dict, not DB
- ❌ Auto-detection - User selects account manually
- ❌ Account discovery - No scraping/API integration
- ❌ Multi-user account sharing
- ❌ Account metadata (bank routing #, last 4 digits)

**Database Schema (Simplified):**
```sql
-- No accounts table! Hardcoded in Python
-- canonicals.account field is just string:
ALTER TABLE canonicals ADD COLUMN account TEXT NOT NULL;
-- Values: "Bank of America Checking" or "Apple Card"
```

**Implementation:**
- Personal: ~15 lines (hardcoded dict)
- Enterprise: ~800 lines (AccountStore table, discovery, sharing)
- **Darwin uses 2% of Enterprise code**

---

## Journey 7: When System Does Counterparty Resolution

### User Story
> Darwin wants to see "Who did I pay?" Summary: "$132.63 total spent at Whole Foods Market this month."

### Kernel Primitives Used

#### 1. CounterpartyStore (Personal Profile - SKIPPED for v1)
```python
# Darwin's v1: NO counterparty entity
# Merchant name IS the counterparty (no normalization to entity)
# Query aggregates by merchant string directly

def get_spending_by_merchant(month: str):
    query = """
        SELECT merchant, SUM(amount) as total
        FROM canonicals
        WHERE strftime('%Y-%m', date) = ?
        AND amount < 0
        GROUP BY merchant
        ORDER BY total
    """
    return db.execute(query, [month]).fetchall()
```

**Decision:** **Skip CounterpartyStore for v1**

**Rationale:**
- Normalizer already cleaned merchant names ("WHOLE FOODS #1234" → "Whole Foods Market")
- Good enough for personal use (95% accuracy)
- CounterpartyStore adds entity resolution (Whole Foods Market → Entity #12345)
- Overkill for 500 tx/month

**What Enterprise Would Use:**
- ✅ CounterpartyStore - Entity table with canonical IDs
- ✅ RelationshipStore - Link merchants to parent entities
- ✅ External data enrichment - Dun & Bradstreet API for business data
- ✅ Fuzzy entity resolution - "Whole Foods" = "Whole Foods Market" = "WFM"

**Implementation:**
- Personal: 0 lines (feature not built, use merchant string directly)
- Enterprise: ~2000 lines (entity resolution + graph)
- **Darwin skips 100% of this feature**

---

### Alternative: Simple Aggregation

```sql
-- Darwin's approach: Aggregate by merchant string
SELECT
    merchant,
    COUNT(*) as transaction_count,
    SUM(amount) as total_spent
FROM canonicals
WHERE strftime('%Y-%m', date) = '2024-10'
  AND amount < 0
GROUP BY merchant
ORDER BY total_spent
LIMIT 10;

-- Results:
-- Whole Foods Market | 8 | -$132.63
-- Starbucks          | 12 | -$69.00
-- Shell Gas Station  | 4 | -$180.00
```

**Good enough for Personal use** ✅

---

## Journey 8: When System Does Clustering (Post-Entity Resolution)

### User Story
> Darwin wants to see related transactions grouped together (e.g., all "groceries" purchases, all "recurring subscriptions").

### Kernel Primitives Used

#### 1. ClusteringEngine (Personal Profile - Simple Categories)
```python
# Simple rule-based categorization (no ML clustering)
CATEGORY_RULES = {
    "Groceries": ["Whole Foods Market", "Safeway", "Trader Joe's"],
    "Dining": ["Starbucks", "Chipotle", "McDonald's"],
    "Gas": ["Shell", "Chevron", "76"],
    "Utilities": ["PG&E", "Comcast"],
    "Subscriptions": ["Netflix", "Spotify", "Apple"]
}

def categorize_transaction(merchant: str) -> str:
    for category, merchants in CATEGORY_RULES.items():
        if any(m.lower() in merchant.lower() for m in merchants):
            return category
    return "Other"

# Applied during normalization
canonical.category = categorize_transaction(canonical.merchant)
```

**Features Used:**
- ✅ Simple rule-based categorization
- ✅ Hardcoded rules (20-30 merchants)
- ✅ Manual category assignment

**Features NOT Used:**
- ❌ Machine learning clustering (K-means, DBSCAN)
- ❌ ClusteringEngine primitive - No clustering algorithm
- ❌ Similarity scores
- ❌ Multi-dimensional clustering (amount + merchant + date)
- ❌ User feedback loop (learn from corrections)

**Implementation:**
- Personal: ~40 lines (hardcoded rules dict)
- Enterprise: ~3000 lines (ML clustering + feature engineering)
- **Darwin uses 1% of Enterprise code**

---

### UI: Category Filter

```typescript
// Simple dropdown filter
function CategoryFilter({ transactions, onFilter }) {
  const categories = [...new Set(transactions.map(t => t.category))];

  return (
    <select onChange={e => onFilter(e.target.value)}>
      <option value="">All Categories</option>
      {categories.map(cat => (
        <option key={cat} value={cat}>{cat}</option>
      ))}
    </select>
  );
}
```

**Good enough for Personal use** ✅

---

## Journey 9: When User Corrects an Error

### User Story
> Darwin notices "Shell Gas Station" was auto-categorized as "Gas" but this purchase was actually snacks from gas station convenience store (should be "Groceries"). He clicks "Edit", changes category to "Groceries", saves.

### Kernel Primitives Used

#### 1. RetroactiveCorrector (Personal Profile - Simple UPDATE)
```python
# Simple SQL UPDATE (no bitemporal tracking)
def update_transaction_category(transaction_id: str, new_category: str):
    db.execute("""
        UPDATE canonicals
        SET category = ?, updated_at = CURRENT_TIMESTAMP
        WHERE id = ?
    """, [new_category, transaction_id])

    # No version history, no audit log
    # Old value lost forever (acceptable for personal use)
```

**Features Used:**
- ✅ Direct UPDATE - Overwrite old value
- ✅ `updated_at` timestamp

**Features NOT Used:**
- ❌ RetroactiveCorrector primitive - No bitemporal
- ❌ Version history - Can't see old value
- ❌ AuditLog - No "who changed what when"
- ❌ Undo/redo - Change is permanent
- ❌ Retroactive recalculation - No dependent values to update
- ❌ Conflict resolution - Single user, no conflicts

**Implementation:**
- Personal: ~10 lines (simple UPDATE)
- Enterprise: ~1800 lines (bitemporal + audit + undo)
- **Darwin uses 0.5% of Enterprise code**

---

### UI: Inline Edit

```typescript
// Simple inline edit
function TransactionRow({ transaction, onUpdate }) {
  const [editing, setEditing] = useState(false);
  const [category, setCategory] = useState(transaction.category);

  if (editing) {
    return (
      <tr>
        <td>{transaction.date}</td>
        <td>{transaction.merchant}</td>
        <td>{transaction.amount}</td>
        <td>
          <select value={category} onChange={e => setCategory(e.target.value)}>
            {CATEGORIES.map(cat => <option key={cat}>{cat}</option>)}
          </select>
        </td>
        <td>
          <button onClick={() => { onUpdate(transaction.id, category); setEditing(false); }}>
            Save
          </button>
        </td>
      </tr>
    );
  }

  return (
    <tr onClick={() => setEditing(true)}>
      <td>{transaction.date}</td>
      <td>{transaction.merchant}</td>
      <td>{transaction.amount}</td>
      <td>{transaction.category}</td>
      <td>—</td>
    </tr>
  );
}
```

**Good enough for Personal use** ✅

---

## Journey 10: When System Connects Transaction to Series

### User Story
> Darwin pays Netflix $15.99 every month. System should recognize this is recurring subscription (same merchant, same amount, monthly frequency).

### Kernel Primitives Used

#### 1. SeriesDetector (Personal Profile - SKIPPED for v1)
```python
# Darwin's v1: NO series detection
# User manually tags recurring transactions with "subscription" category
# No automatic detection of recurring patterns
```

**Decision:** **Skip SeriesDetector for v1**

**Rationale:**
- Series detection requires pattern matching (same merchant + amount + frequency)
- Complex edge cases: Amount changes ($15.99 → $16.99), skipped months, billing date shifts
- Low value for 500 tx/month (user knows which are subscriptions)
- Manual tagging simpler

**What Enterprise Would Use:**
- ✅ SeriesDetector - Detect recurring patterns
- ✅ Frequency analysis - Monthly, weekly, annual
- ✅ Tolerance thresholds - Amount ±$1, date ±3 days
- ✅ Series lifecycle - Start date, end date, paused periods
- ✅ Forecasting - Predict next occurrence

**Implementation:**
- Personal: 0 lines (not built)
- Enterprise: ~2500 lines (pattern matching + forecasting)
- **Darwin skips 100% of this feature**

---

### Alternative: Manual Tagging

```python
# User tags subscriptions manually
def tag_as_subscription(transaction_id: str):
    db.execute("""
        UPDATE canonicals
        SET tags = 'subscription'
        WHERE id = ?
    """, [transaction_id])

# Filter view: Show all subscriptions
query = """
    SELECT * FROM canonicals
    WHERE tags LIKE '%subscription%'
    ORDER BY date DESC
"""
```

**Good enough for Personal use** ✅

---

## Journey 11: When System Categorizes for USA Taxes

### User Story
> Darwin needs year-end tax summary: "Show me all deductible business expenses (meals, mileage, office supplies)." System should categorize transactions by IRS tax categories.

### Kernel Primitives Used

#### 1. TaxCategoryMapper (Personal Profile - Simple Rules)
```python
# Hardcoded IRS tax category mappings
TAX_CATEGORY_RULES = {
    "Meals & Entertainment (50% deductible)": ["Starbucks", "Chipotle", "restaurant"],
    "Office Supplies (100% deductible)": ["Staples", "Office Depot", "Amazon"],
    "Mileage (IRS rate $0.67/mi)": ["Shell", "Chevron", "gas station"],
    "Software & Subscriptions (100% deductible)": ["Adobe", "GitHub", "AWS"],
    "Non-deductible": ["Whole Foods", "grocery", "personal"]
}

def assign_tax_category(merchant: str, category: str) -> str:
    for tax_cat, keywords in TAX_CATEGORY_RULES.items():
        if any(kw.lower() in merchant.lower() for kw in keywords):
            return tax_cat
    return "Non-deductible"

# Applied on-demand (not stored)
def get_tax_summary(year: int):
    transactions = get_transactions_for_year(year)

    summary = {}
    for tx in transactions:
        tax_cat = assign_tax_category(tx.merchant, tx.category)
        summary[tax_cat] = summary.get(tax_cat, 0) + tx.amount

    return summary
```

**Features Used:**
- ✅ Hardcoded tax rules (10-15 categories)
- ✅ On-demand calculation (not stored in DB)
- ✅ Simple text matching

**Features NOT Used:**
- ❌ TaxCategoryMapper table - No persistent tax categories
- ❌ Multi-jurisdiction support - USA only
- ❌ Tax form generation - No PDF exports (user copies numbers manually)
- ❌ Receipt attachment - No linking receipts to transactions
- ❌ Audit trail - No IRS-compliant logs
- ❌ Tax rule updates - Hardcoded (user updates code if IRS rules change)

**Implementation:**
- Personal: ~60 lines (hardcoded rules + summary)
- Enterprise: ~4000 lines (multi-jurisdiction + form generation + audit)
- **Darwin uses 1.5% of Enterprise code**

---

### UI: Tax Summary Report

```typescript
// Simple summary table
function TaxSummary({ year }) {
  const [summary, setSummary] = useState({});

  useEffect(() => {
    fetch(`/api/tax-summary/${year}`)
      .then(res => res.json())
      .then(data => setSummary(data));
  }, [year]);

  return (
    <table>
      <thead>
        <tr>
          <th>Tax Category</th>
          <th>Total</th>
        </tr>
      </thead>
      <tbody>
        {Object.entries(summary).map(([category, total]) => (
          <tr key={category}>
            <td>{category}</td>
            <td>${Math.abs(total).toFixed(2)}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**Good enough for Personal use** ✅

---

## Journey 12: When System Links FX Fee Transaction

### User Story
> Darwin travels to Europe, uses credit card. Statement shows 2 transactions: "€50.00 Restaurant in Paris" and "$2.50 Foreign Transaction Fee". System should link fee to original transaction.

### Kernel Primitives Used

#### 1. FXFeeLinker (Personal Profile - SKIPPED for v1)
```python
# Darwin's v1: NO automatic FX fee linking
# Rare edge case (Darwin travels 1-2 times/year)
# User manually identifies fees (search for "Foreign Transaction Fee")
```

**Decision:** **Skip FXFeeLinker for v1**

**Rationale:**
- FX fee linking requires complex heuristics (fee appears 1-3 days after charge, amount is ~3% of original)
- Rare occurrence for Darwin (<10 fees/year)
- Manual identification trivial (search for "Foreign")
- Not worth building automation

**What Enterprise Would Use:**
- ✅ FXFeeLinker - Detect fee patterns (amount = 3% of parent tx)
- ✅ Time window matching - Fee within 3 days of charge
- ✅ Currency conversion - Store original currency + USD equivalent
- ✅ Multi-currency support - CurrencyConverter primitive
- ✅ FX rate history - ExchangeRateProvider integration

**Implementation:**
- Personal: 0 lines (not built)
- Enterprise: ~1200 lines (pattern matching + multi-currency)
- **Darwin skips 100% of this feature**

---

### Alternative: Manual Search

```sql
-- Darwin's approach: Search for FX fees manually
SELECT * FROM canonicals
WHERE merchant LIKE '%Foreign%' OR merchant LIKE '%Transaction Fee%'
ORDER BY date DESC;

-- Results:
-- 2024-08-15 | Foreign Transaction Fee | -$2.50 | Fees
-- 2024-08-12 | Foreign Transaction Fee | -$1.75 | Fees

-- User manually notes: "$2.50 fee relates to €50 Paris restaurant (Aug 13)"
```

**Good enough for Personal use** ✅

---

## Final Summary: Feature Comparison

| Journey | Personal Features | Enterprise Features | % Used |
|---------|-------------------|---------------------|--------|
| 1. Upload | Hash, store, metadata | + S3, encryption, virus scan, metrics | 7% |
| 2. Parse | Hardcoded parser, basic extract | + Registry, versioning, canary deploys | 6% |
| 3. View OL | Simple SELECT, table | + Pagination, filters, virtual scroll | 10% |
| 4. View RL | Rule-based normalize, basic UI | + ML, fuzzy match, APIs, bitemporal | 8% |
| 5. Duplicates | **SKIPPED** (manual) | Auto fuzzy detection, confidence scores | 0% |
| 6. Accounts | Hardcoded dict (2 accounts) | AccountStore table, discovery, sharing | 2% |
| 7. Counterparty | **SKIPPED** (use merchant string) | Entity resolution, relationship graph | 0% |
| 8. Clustering | Hardcoded rules (20 merchants) | ML clustering, feature engineering | 1% |
| 9. Corrections | Simple UPDATE | Bitemporal, audit log, undo/redo | 0.5% |
| 10. Series | **SKIPPED** (manual tags) | Pattern detection, forecasting | 0% |
| 11. Tax | Hardcoded IRS rules (15 categories) | Multi-jurisdiction, form generation | 1.5% |
| 12. FX Fees | **SKIPPED** (manual search) | Auto-linking, multi-currency | 0% |

---

## Overall Statistics

### Code Comparison

| Component | Personal LOC | Enterprise LOC | % Used |
|-----------|--------------|----------------|--------|
| **Journey 1-4 (Core)** | 1,220 | 18,000 | 7% |
| **Journey 5-8 (Resolution)** | 75 | 8,700 | 0.9% |
| **Journey 9-12 (Advanced)** | 70 | 9,000 | 0.8% |
| **TOTAL** | **1,365** | **35,700** | **3.8%** |

### Features Built vs Skipped

**Personal (Darwin):**
- ✅ Built: 8 features (upload, parse, view OL/RL, normalize, accounts, categories, tax, corrections)
- ❌ Skipped: 4 features (duplicates, counterparty, series, FX fees)
- **Build ratio:** 67% (8/12)

**Enterprise:**
- ✅ Built: 12 features (all)
- ❌ Skipped: 0 features
- **Build ratio:** 100% (12/12)

---

## Key Insights

### 1. Hardcoded is Appropriate for Static Context
- Darwin has 2 accounts (BoFA, Apple Card) → Hardcoded dict ✅
- Enterprise has 1000s of accounts → AccountStore table required ✅

### 2. Skip Rare Features (YAGNI Principle)
- Duplicates: <5/year → Manual handling ✅
- FX fees: <10/year → Manual search ✅
- Series: User knows subscriptions → Manual tags ✅

### 3. Simple Rules Beat ML at Small Scale
- 500 tx/month → Hardcoded rules (20 merchants) work perfectly
- 10M tx/month → ML required (can't hardcode 100K+ merchants)

### 4. Same Interfaces, Different Implementations
- **Personal:** Hardcoded dicts, simple functions, direct SQL
- **Enterprise:** Tables, algorithms, APIs, background jobs
- **Both use:** Same primitive names (Parser, Normalizer, AccountStore concepts)

### 5. Darwin Uses 3.8% of Enterprise Code
- **1,365 lines** (Personal) vs **35,700 lines** (Enterprise)
- Same functionality for his use case (500 tx/month)
- Enterprise complexity unnecessary at his scale

---

## Conclusion

The finance-app-kernel demonstrates **scale-appropriate complexity**:

- **Personal (Darwin):** Minimalist implementation, skip rare features, hardcode stable data
- **Small Business:** Add configuration, basic automation, multi-user support
- **Enterprise:** Full feature set, ML/AI, compliance, high availability

**The same kernel interfaces work at all scales.** Darwin cherry-picks 4% of the code. Enterprise uses 100%. Both succeed at their respective scales.

This is the essence of **literate programming + simplicity profiles**: Build abstractions that scale from 1 user to 1M users, but let each implementer choose their complexity level.
