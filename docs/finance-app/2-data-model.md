# Data Model (Minimal)

> **Purpose:** Define the core data structures needed for the finance app (simple, practical, no enterprise complexity)

---

## Design Philosophy

**Start Simple, Grow Later**

This data model represents what we need NOW for 500 transactions/month, not what we might need for 10M transactions/month.

Key decisions:
- ❌ **No bitemporal tracking** (transaction_time/valid_time) - can add later if needed
- ❌ **No provenance ledger** (full audit trail) - simple audit_log is sufficient
- ❌ **No sharding keys** (partition_id, shard_id) - single SQLite database
- ✅ **Simple foreign keys** (account_id, category_id, counterparty_id)
- ✅ **Straightforward types** (TEXT, REAL, INTEGER, DATETIME)

---

## Core Entities

### 1. Transaction (8 fields)

**Purpose:** Represents a single financial transaction

```sql
CREATE TABLE transactions (
  id           TEXT PRIMARY KEY,      -- TX_abc123
  date         DATE NOT NULL,         -- 2024-10-15
  amount       REAL NOT NULL,         -- -87.43 (negative = expense)
  merchant     TEXT NOT NULL,         -- "Whole Foods Market"
  category_id  TEXT,                  -- FK to categories.id
  account_id   TEXT NOT NULL,         -- FK to accounts.id
  notes        TEXT,                  -- User-added notes
  tags         TEXT,                  -- Comma-separated: "groceries,weekly"

  created_at   DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Field Decisions:**

- **id**: Generated UUID with TX_ prefix (human-readable in logs)
- **date**: DATE only (no time) - bank statements don't include time
- **amount**: Signed decimal (negative = expense, positive = income)
- **merchant**: Normalized merchant name (not raw PDF description)
- **category_id**: FK to categories (NULL = uncategorized)
- **account_id**: Required - every transaction belongs to an account
- **notes**: User can add context (e.g., "split with roommate")
- **tags**: Flexible tagging system (e.g., "tax-deductible", "reimbursable")

**What's NOT in this table:**
- ❌ Raw PDF data (stored separately in observations)
- ❌ Normalization history (stored separately in audit_log)
- ❌ Currency code (assumed USD for v1)
- ❌ Exchange rate (can add later for multi-currency)

---

### 2. Category (3 fields)

**Purpose:** Organize transactions into spending categories

```sql
CREATE TABLE categories (
  id          TEXT PRIMARY KEY,      -- CAT_groceries
  name        TEXT NOT NULL UNIQUE,  -- "Groceries"
  parent_id   TEXT,                  -- FK to categories.id (for subcategories)
  color       TEXT,                  -- "#4CAF50" (hex color for UI)

  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Hierarchical Structure:**

```
Groceries (CAT_groceries)
  ├─ Supermarkets (CAT_groceries_supermarkets)
  ├─ Farmers Markets (CAT_groceries_farmers)
  └─ Specialty Stores (CAT_groceries_specialty)

Food & Dining (CAT_food)
  ├─ Restaurants (CAT_food_restaurants)
  ├─ Coffee Shops (CAT_food_coffee)
  └─ Fast Food (CAT_food_fastfood)
```

**Seed Categories (20-30 common categories):**
```
- Groceries
- Rent/Mortgage
- Utilities (Electric, Water, Gas)
- Transportation (Gas, Parking, Uber)
- Food & Dining
- Entertainment (Movies, Streaming, Concerts)
- Healthcare (Doctor, Pharmacy, Insurance)
- Shopping (Clothing, Electronics, Home)
- Travel (Flights, Hotels, Vacation)
- Personal Care (Haircut, Gym)
- Insurance (Car, Home, Life)
- Taxes (Federal, State, Property)
- Charitable Donations
- Savings & Investments
- Other
```

**Max depth:** 3 levels (Category → Subcategory → Sub-subcategory)

---

### 3. Account (4 fields)

**Purpose:** User's financial accounts (checking, savings, credit cards)

```sql
CREATE TABLE accounts (
  id           TEXT PRIMARY KEY,      -- ACC_chase_checking
  name         TEXT NOT NULL,         -- "Chase Checking"
  institution  TEXT NOT NULL,         -- "Chase Bank"
  last4        TEXT NOT NULL,         -- "1234" (last 4 digits)
  type         TEXT NOT NULL,         -- checking, savings, credit_card

  active       BOOLEAN DEFAULT TRUE,  -- Soft delete
  created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Account Types:**
- `checking` - Bank checking account
- `savings` - Bank savings account
- `credit_card` - Credit card
- `investment` - Investment/brokerage account (future)
- `loan` - Loan/mortgage (future)

**Why last4?**
- Bank statements show "Account: ****1234"
- Helps user identify which account without showing full number
- Used for account resolution when importing statements

**Active flag:**
- Soft delete - never physically delete accounts (historical transactions reference them)
- User can "archive" old accounts (active=false)
- Archived accounts hidden from UI but preserved in database

---

## Supporting Entities

### 4. Counterparty (simplified)

**Purpose:** Merchants, employers, other entities user transacts with

```sql
CREATE TABLE counterparties (
  id            TEXT PRIMARY KEY,      -- CP_wholefoods
  canonical_name TEXT NOT NULL UNIQUE, -- "Whole Foods Market"
  type          TEXT NOT NULL,         -- merchant, employer, person, other

  created_at    DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE counterparty_aliases (
  counterparty_id TEXT NOT NULL,       -- FK to counterparties.id
  alias           TEXT NOT NULL,       -- "WHOLE FOODS MKT #1234"

  PRIMARY KEY (counterparty_id, alias)
);
```

**Why separate aliases table?**
- One merchant can have many variations in bank statements
- "WHOLE FOODS MARKET #1234 SF", "WFM", "WHOLE FOODS MKT" → all same counterparty
- Allows fuzzy matching without duplicating data

---

### 5. Series (recurring payments)

**Purpose:** Track recurring transactions (subscriptions, rent, etc.)

```sql
CREATE TABLE series (
  id          TEXT PRIMARY KEY,      -- SER_netflix
  name        TEXT NOT NULL,         -- "Netflix Subscription"
  amount      REAL NOT NULL,         -- 14.99
  tolerance   REAL DEFAULT 1.0,      -- ± $1 variance allowed
  frequency   TEXT NOT NULL,         -- monthly, weekly, yearly
  day_of_month INTEGER,              -- 15 (for monthly)
  counterparty_id TEXT,              -- FK to counterparties.id
  category_id TEXT,                  -- FK to categories.id

  active      BOOLEAN DEFAULT TRUE,
  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE series_instances (
  id              TEXT PRIMARY KEY,  -- SI_netflix_oct2024
  series_id       TEXT NOT NULL,     -- FK to series.id
  transaction_id  TEXT NOT NULL,     -- FK to transactions.id
  expected_date   DATE NOT NULL,     -- 2024-10-15
  actual_date     DATE,              -- 2024-10-16 (1 day late)
  expected_amount REAL NOT NULL,     -- 14.99
  actual_amount   REAL,              -- 15.99 (price increase)
  variance        REAL,              -- 1.00
  status          TEXT NOT NULL,     -- matched, variance, missing

  created_at      DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Status values:**
- `matched` - Transaction found, amount within tolerance
- `variance` - Transaction found, amount outside tolerance (flag for review)
- `missing` - Expected transaction not found (cancelled or pending)

---

### 6. Categorization Rules (YAML config, not database)

**Purpose:** Define how to auto-categorize transactions

**File: `config/categorization_rules.yaml`**

```yaml
rules:
  - pattern: "WHOLE FOODS"
    category: "Groceries"
    confidence: 0.9

  - pattern: "NETFLIX"
    category: "Entertainment"
    confidence: 0.95

  - pattern: "SHELL|CHEVRON|76"
    category: "Transportation"
    confidence: 0.85

  - pattern: "STARBUCKS|PEETS|PHILZ"
    category: "Food & Dining"
    subcategory: "Coffee Shops"
    confidence: 0.9

  - pattern: "UBER|LYFT"
    category: "Transportation"
    subcategory: "Rideshare"
    confidence: 0.95
```

**Why YAML instead of database?**
- Easier to version control (git)
- Easier to bulk edit (text editor)
- Easier to share/import/export
- Can load into memory at startup (50-100 rules = <1MB)

**Can be moved to database later if:**
- User needs web UI for rule editing
- Multiple users need separate rule sets
- Rules exceed 1000+ (performance consideration)

---

## Audit & Metadata

### 7. Audit Log (simple)

**Purpose:** Track user actions for accountability (not full provenance)

```sql
CREATE TABLE audit_log (
  id           TEXT PRIMARY KEY,      -- LOG_abc123
  action       TEXT NOT NULL,         -- "category_changed", "transaction_deleted"
  entity_type  TEXT NOT NULL,         -- "transaction", "category", "account"
  entity_id    TEXT NOT NULL,         -- TX_abc123, CAT_groceries
  old_value    TEXT,                  -- JSON: {"category_id": "CAT_shopping"}
  new_value    TEXT,                  -- JSON: {"category_id": "CAT_books"}
  user_id      TEXT,                  -- "darwin" (for multi-user future)
  timestamp    DATETIME DEFAULT CURRENT_TIMESTAMP,

  INDEX idx_entity (entity_type, entity_id),
  INDEX idx_timestamp (timestamp)
);
```

**What gets logged:**
- Category changes
- Transaction deletions
- Account edits
- Manual corrections

**What does NOT get logged (yet):**
- Every database query (too verbose)
- Every page view (not needed for v1)
- System operations (parsing, normalization) - separate logs for this

---

### 8. Upload Records (state machine)

**Purpose:** Track PDF uploads and processing status

```sql
CREATE TABLE upload_records (
  id              TEXT PRIMARY KEY,      -- UL_abc123
  filename        TEXT NOT NULL,         -- "bofa-statement-october.pdf"
  file_size       INTEGER NOT NULL,      -- 1245678 (bytes)
  file_hash       TEXT NOT NULL UNIQUE,  -- SHA-256 hash (for deduplication)
  status          TEXT NOT NULL,         -- queued, parsing, parsed, error
  uploaded_by     TEXT,                  -- "darwin"
  uploaded_at     DATETIME DEFAULT CURRENT_TIMESTAMP,
  processed_at    DATETIME,
  error_message   TEXT,

  observations_count INTEGER DEFAULT 0,  -- How many raw transactions extracted

  INDEX idx_status (status),
  INDEX idx_hash (file_hash)
);
```

**Status state machine:**
```
queued → parsing → parsed → (complete)
          ↓
        error
```

**Deduplication:**
- If file_hash exists → return 409 Conflict before parsing
- Prevents duplicate imports

---

## Relationships Diagram

```
┌─────────────┐
│  uploads    │
└─────────────┘
       │
       ↓ (creates)
┌─────────────┐
│observations │ (raw data from PDF - separate table)
└─────────────┘
       │
       ↓ (normalizes to)
┌─────────────┐        ┌─────────────┐
│transactions │───────→│  accounts   │
└─────────────┘        └─────────────┘
       │                      ↑
       ↓                      │
┌─────────────┐               │
│ categories  │               │
└─────────────┘               │
       ↑                      │
       │                      │
┌─────────────┐        ┌─────────────┐
│counterparties│       │   series    │
└─────────────┘        └─────────────┘
       │                      │
       ↓                      ↓
┌─────────────┐        ┌─────────────┐
│   aliases   │        │  instances  │
└─────────────┘        └─────────────┘
```

---

## What's Deliberately Excluded (Can Add Later)

**Bitemporal Tracking:**
- No `transaction_time` / `valid_time` columns
- Simple `updated_at` timestamp is sufficient for v1
- Can add later if tax audits require "what did I know on date X?"

**Multi-Currency:**
- No `currency_code` field
- No `exchange_rate` field
- Assumed USD for v1
- Can add later if user travels internationally frequently

**Provenance Ledger:**
- No append-only event store
- Simple audit_log with UPDATE capability is sufficient
- Can add later if compliance requires immutable audit trail

**Tagging System:**
- Simple comma-separated `tags` field
- No separate `tags` table with many-to-many relationship
- Can normalize later if tag management becomes complex

**Attachments:**
- No `attachments` table (receipts, invoices)
- User can link notes like "receipt in Dropbox: /tax-docs/2024/receipt-123.pdf"
- Can add later if receipt management becomes important

**Budget Tracking:**
- No `budgets` table
- Can calculate spending by category manually
- Can add later if user wants alerts ("spent 90% of grocery budget")

---

## Database Indexes

**Critical for Performance:**

```sql
-- Transactions
CREATE INDEX idx_transactions_date ON transactions(date);
CREATE INDEX idx_transactions_account ON transactions(account_id);
CREATE INDEX idx_transactions_category ON transactions(category_id);
CREATE INDEX idx_transactions_date_account ON transactions(date, account_id);

-- Audit Log
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_timestamp ON audit_log(timestamp DESC);

-- Upload Records
CREATE INDEX idx_uploads_status ON upload_records(status);
CREATE INDEX idx_uploads_hash ON upload_records(file_hash);
```

**Why these indexes?**
- `date` - Most queries filter by date range
- `account_id` - "Show transactions for Chase Checking"
- `category_id` - "Show all Grocery spending"
- `date + account_id` - Compound index for common combo query
- `status` - Worker queries for `WHERE status = 'queued'`

---

## Data Sizes (500 tx/month)

**Storage Estimates:**

| Table | Rows/Year | Size/Row | Total/Year |
|-------|-----------|----------|------------|
| transactions | 6,000 | ~200 bytes | 1.2 MB |
| categories | 30 | ~100 bytes | 3 KB |
| accounts | 6 | ~100 bytes | 600 bytes |
| counterparties | 200 | ~150 bytes | 30 KB |
| series | 10 | ~200 bytes | 2 KB |
| upload_records | 120 | ~300 bytes | 36 KB |
| audit_log | 500 | ~400 bytes | 200 KB |

**Total Database Size:** ~1.5 MB/year

**With 5 years of data:** ~7.5 MB (SQLite easily handles this)

**PDFs:** 120 PDFs/year × 2 MB = 240 MB/year × 5 years = 1.2 GB

**Total Storage:** ~1.2 GB for 5 years of data (fits on any laptop)

---

## Migration Strategy

**When to upgrade to PostgreSQL:**
- Transactions exceed 50,000 (would take 8+ years at current rate)
- Multiple concurrent users
- Need for row-level security
- Geographic distribution (read replicas)

**When to add bitemporal tracking:**
- Tax audit requires "what did I know on date X?" queries
- Regulatory compliance (GDPR, SOX)
- Need to reconstruct historical states

**When to separate observations table:**
- Need to re-process historical data with new parsers
- Want to A/B test different normalization rules

---

## Summary

**This data model is:**
- ✅ Simple (8 tables, ~50 columns total)
- ✅ Sufficient for 500 tx/month
- ✅ SQLite-compatible
- ✅ Extensible (can add fields later)

**This data model is NOT:**
- ❌ Enterprise-scale (no sharding, no partitioning)
- ❌ Multi-tenant (no tenant_id isolation)
- ❌ Compliance-ready (no immutable audit trail)

**Key Insight:**
Start with simple schema that solves real needs. Add complexity only when pain points emerge, not in anticipation of theoretical problems.
