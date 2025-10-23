# 📋 CANONICAL VERTICAL LIST

> **⚠️ SINGLE SOURCE OF TRUTH** — This is the official list of all verticals in the Truth Construction Primitives project. All references to vertical numbers MUST use this document as the canonical source.

---

## 🎯 Total: 23 Verticals

| Group | Count | Verticals |
|-------|-------|-----------|
| **1. Upload & Ingestion** | 3 | 1.1 - 1.3 |
| **2. Exploration & Visualization** | 3 | 2.1 - 2.3 |
| **3. Registries (Finance Domain)** | 9 | 3.1 - 3.9 |
| **4. Derivatives & Operativa** | 3 | 4.1 - 4.3 |
| **5. Governance & Meta** | 5 | 5.1 - 5.5 |
| **TOTAL** | **23** | |

---

## 📦 Group 1: Upload & Ingestion

### **1.1 Upload Flow**
**Status:** ✅ Complete
**Full Name:** Upload Flow (artefactos / fuentes de verdad)
**Additions:** + File naming & dedupe
**Spec:** [docs/verticals/1.1-upload-flow.md](docs/verticals/1.1-upload-flow.md)

**Key Primitives:**
- StorageEngine (content-addressable storage)
- ProvenanceLedger (append-only audit trail)
- FileArtifact (metadata wrapper)
- HashCalculator (SHA-256 streaming)
- UploadRecord (state machine)

**Schemas:** upload-record, provenance-entry, upload-error-response
**ADRs:** 0001 (canonical ID), 0002 (state machine), 0003 (runner/coordinator)

---

### **1.2 Extraction**
**Status:** ✅ Complete
**Full Name:** Document Processing / Extraction (parser → observations)
**Additions:** + States & retries
**Spec:** [docs/verticals/1.2-extraction.md](docs/verticals/1.2-extraction.md)

**Key Primitives:**
- Parser (universal interface for document extraction)
- ObservationStore (raw observation storage)
- ParserRegistry (pluggable parser discovery)
- ParseLog (execution log)

**Schemas:** observation-transaction, parse-log
**ADRs:** 0004 (raw-first extraction)

---

### **1.3 Normalization**
**Status:** ✅ Complete
**Full Name:** Normalization (observations → canonical facts)
**Additions:** + States
**Spec:** [docs/verticals/1.3-normalization.md](docs/verticals/1.3-normalization.md)

**Key Primitives:**
- Normalizer (raw → canonical transformation)
- ValidationEngine (rule-based validation)
- CanonicalStore (validated record storage)
- NormalizationRuleSet (externalized rules)
- NormalizationLog (execution log)

**Schemas:** canonical-transaction, normalization-log
**ADRs:** 0005 (configurable normalization rules)

**Out of scope:** Multi-account reconciliation, transfer linking (moved to 3.5 or 3.9), analytics (Group 2)

---

## 📊 Group 2: Exploration & Visualization

### **2.1 Transaction List View**
**Status:** ✅ Complete
**Full Name:** Transaction List View
**Additions:** + Pagination & Indexing
**Spec:** [docs/verticals/2.1-transaction-list-view.md](docs/verticals/2.1-transaction-list-view.md)

**Primitives Delivered:**
- TransactionQuery (universal query builder with filter/sort/pagination)
- PaginationEngine (cursor-based pagination - keyset pagination)
- IndexStrategy (database index suggestion based on query patterns)
- ExportEngine (CSV/PDF/Excel export generation)
- TransactionTable (IL - data grid component with sorting/pagination)

**Schemas:** transaction-query-response.schema.json
**UX Flow:** [2.1-transaction-list-experience.md](docs/ux-flows/2.1-transaction-list-experience.md)

**Delivered:**
- View canonical transactions with server-side filtering
- Filter by date, account, category, amount, search text
- Sort by any column (date, amount, merchant, etc.)
- Cursor-based pagination (stable under concurrent inserts)
- Summary metrics (income, expenses, net) for filtered results
- Export to CSV/PDF with metadata
- Responsive table UI with loading skeletons
- Accessibility support (keyboard nav, screen readers)

---

### **2.2 OL Exploration**
**Status:** 📝 Pending
**Full Name:** OL Exploration (drill-down, decisions, provenance)
**Spec:** TBD

**Expected Use Cases:**
- Drill down from canonical → observation → upload artifact
- View normalization decisions (why was this categorized as X?)
- Trace provenance chain (who uploaded, when, from where)

---

### **2.3 Finance Dashboard**
**Status:** 📝 Pending
**Full Name:** Finance Dashboard (saved views)
**Additions:** + Exports & Snapshots
**Spec:** TBD

**Expected Use Cases:**
- Pre-configured views (monthly spending, income vs expenses)
- Custom dashboards (user-defined metrics)
- Saved filters and layouts
- Snapshot exports (PDF reports for accountant)

---

## 🗂️ Group 3: Registries (Finance Domain)

### **3.1 Account**
**Status:** 📝 Pending
**Full Name:** Account (closed registry)
**Spec:** TBD

**Expected:**
- Closed registry = fixed list of user's accounts
- Examples: BofA Debit, BofA Credit, Apple Card, Scotia, Wise, Stripe
- Attributes: account_id, name, type (checking, credit, etc.), currency, institution

---

### **3.2 Counterparty**
**Status:** 📝 Pending
**Full Name:** Counterparty (open registry)
**Spec:** TBD

**Expected:**
- Open registry = dynamic, grows as new merchants/people appear
- Examples: OpenAI, HubSpot, Uber, Diana de la Tejera
- Attributes: counterparty_id, canonical_name, aliases, type (merchant, person, business)

---

### **3.3 Series**
**Status:** 📝 Pending
**Full Name:** Series (closed registry)
**Spec:** TBD

**Expected:**
- Closed registry = user-defined recurring payment series
- Examples: "OpenAI Subscription", "Rent", "Electricity (CFE)"
- Attributes: series_id, name, expected_amount, frequency, account

---

### **3.4 Tax Categorization**
**Status:** 📝 Pending
**Full Name:** Tax Categorization
**Spec:** TBD

**Expected:**
- Dual tax jurisdiction support (USA + Mexico)
- Deductible tagging (100%, 50%, 0%)
- Schedule C categories (USA), SAT categories (Mexico)
- Factura tracking (Mexico)

---

### **3.5 Relationships**
**Status:** 📝 Pending
**Full Name:** Relationships (txn ↔ txn)
**Additions:** + fx_conversion
**Spec:** TBD

**Expected:**
- Link paired transactions (BofA → Wise, Wise → Scotia)
- Foreign exchange conversion tracking
- Transfer detection and reconciliation
- **NOTE:** This likely covers the "Transfer Linking" concept previously mentioned as "1.4"

---

### **3.6 Unit**
**Status:** 📝 Pending
**Full Name:** Unit (normalización)
**Additions:** + currency/date rules
**Spec:** TBD

**Expected:**
- Currency normalization (USD, MXN → base currency)
- Date format normalization (MM/DD/YYYY vs DD/MM/YYYY)
- Amount precision rules
- Exchange rate sources

---

### **3.7 Parser**
**Status:** 📝 Pending
**Full Name:** Parser (registry)
**Spec:** TBD

**Expected:**
- Registry of available parsers (BofA PDF, Apple Card CSV, Scotia PDF, etc.)
- Parser versioning
- Parser capabilities (what fields can it extract)

---

### **3.8 Cluster Rules**
**Status:** 📝 Pending
**Full Name:** Cluster Rules
**Spec:** TBD

**Expected:**
- Merchant name normalization rules ("UBER EATS PENDING" → "Uber Eats")
- Clustering similar transactions
- Auto-categorization rules

---

### **3.9 Reconciliation Strategies**
**Status:** 📝 Pending
**Full Name:** Reconciliation Strategies
**Spec:** TBD

**Expected:**
- Fuzzy matching algorithms (amount ±$1, date ±2 days)
- Confidence scoring
- Auto-suggest vs auto-link thresholds
- **NOTE:** This may also cover transfer reconciliation aspects

---

## 📈 Group 4: Derivatives & Operativa

### **4.1 Reminders**
**Status:** 📝 Pending
**Full Name:** Reminders
**Spec:** TBD

**Expected:**
- Alert if recurring payment didn't occur
- Alert if balance below threshold
- Alert if large expense detected
- Due date reminders (credit card payments)

---

### **4.2 Forecast**
**Status:** 📝 Pending
**Full Name:** Forecast
**Spec:** TBD

**Expected:**
- Income/expense projections
- Cash runway calculation
- Goal tracking (monthly income targets)
- Burn rate analysis

---

### **4.3 Corrections Flow**
**Status:** 📝 Pending
**Full Name:** Corrections Flow
**Additions:** + override precedence & audit por campo
**Spec:** TBD

**Expected:**
- Manual corrections to categorization
- Override normalization decisions
- Audit trail per field (who changed, when, why)
- Precedence rules (manual override > rule > default)

---

## 🏛️ Group 5: Governance & Meta

### **5.1 Provenance Ledger**
**Status:** 📝 Pending
**Full Name:** Provenance Ledger
**Additions:** + bitemporal model
**Spec:** TBD

**Expected:**
- Bitemporal tracking (transaction time vs valid time)
- Immutable audit trail
- Retroactive corrections handling
- "As of" queries (what did we know on date X?)

**NOTE:** Basic provenance implemented in 1.1, this is the advanced version

---

### **5.2 Schema Registry**
**Status:** 📝 Pending
**Full Name:** Schema Registry (Observation / Canonical)
**Additions:** + versioning & migrations
**Spec:** TBD

**Expected:**
- Schema versioning (observation-transaction-v1, v2, etc.)
- Migration strategies
- Backward compatibility
- Schema evolution rules

---

### **5.3 Rule Performance / Logs**
**Status:** 📝 Pending
**Full Name:** Rule Performance / Logs
**Additions:** + parser/queue metrics
**Spec:** TBD

**Expected:**
- Parser execution metrics (latency, success rate)
- Normalization rule performance
- Queue depth and processing times
- Error rates and retry statistics

---

### **5.4 Security & Access**
**Status:** 📝 Pending
**Full Name:** Security & Access (PII, roles)
**Spec:** TBD

**Expected:**
- PII handling (mask account numbers, SSN)
- Role-based access control (owner, accountant read-only)
- Data encryption at rest and in transit
- Audit logs for access

---

### **5.5 Public API Contracts**
**Status:** 📝 Pending
**Full Name:** Public API Contracts (si aplica)
**Spec:** TBD

**Expected:**
- REST API for external integrations
- Webhook support
- API versioning
- Rate limiting and authentication

**NOTE:** User unsure if this evolved/is still needed

---

## 📊 Progress Tracker

| Status | Count | Verticals |
|--------|-------|-----------|
| ✅ Complete | 4 | 1.1, 1.2, 1.3, 2.1 |
| 📝 Pending | 19 | 2.2-2.3, 3.1-3.9, 4.1-4.3, 5.1-5.5 |
| **TOTAL** | **23** | |

**Completion:** 17% (4/23)

**Next up:** 2.2 OL Exploration

---

## 🔒 Change Control

**This document is the single source of truth for vertical numbering.**

### When adding a new vertical:
1. ✅ Update this document FIRST
2. ✅ Get user confirmation
3. ✅ Then create the vertical spec
4. ✅ Update progress tracker

### When referencing a vertical in docs:
- ✅ Always check this document for correct number
- ✅ Use full name: "2.1 Transaction List View" not just "2.1"
- ✅ Link to this document in README

### Obsolete references:
- ❌ "1.4 Transfer Linking" — This concept moved to 3.5 or 3.9, NOT a separate vertical

---

## ❓ Open Questions

1. **Total count:** This list contains 23 verticals, but project was originally scoped as 24. Is one missing, or was the original count incorrect?

2. **3.5 vs 3.9:** Does "Transfer Linking" belong in 3.5 (Relationships) or 3.9 (Reconciliation Strategies), or is it split between both?

3. **5.5 Public API:** User mentioned "no se si esta evoluciono" — confirm if this vertical is still in scope.

---

*Last updated: 2025-05-23*
*Maintained by: Eugenio Castro Garza*
