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
**Status:** ✅ Complete
**Full Name:** OL Exploration (drill-down, decisions, provenance)
**Spec:** [docs/verticals/2.2-ol-exploration.md](docs/verticals/2.2-ol-exploration.md)

**Primitives Delivered:**
- ProvenanceTracer (OL) - Trace lineage from canonical → observation → artifact
- DecisionExplainer (OL) - Explain normalization decisions with rules and confidence
- ArtifactRetriever (OL) - Retrieve artifacts with signed URLs and access control
- DrillDownPanel (IL) - Slide-in panel for complete drill-down view
- ProvenanceTimeline (IL) - Visual timeline for audit trail

**Schemas:** drill-down-response.schema.json, decision-explanation.schema.json
**UX Flow:** [2.2-ol-exploration-experience.md](docs/ux-flows/2.2-ol-exploration-experience.md)

**Delivered:**
- Complete data lineage drill-down (canonical → observation → artifact)
- Normalization decision explanations (rules, confidence, alternatives)
- Provenance timeline visualization (upload → parse → normalize)
- Original artifact viewing (PDF viewer with signed URLs)
- Access control enforcement (user owns transaction)
- Responsive UI (slide-in panel desktop, full-screen mobile)
- Keyboard navigation and accessibility support

---

### **2.3 Finance Dashboard**
**Status:** ✅ Complete
**Full Name:** Finance Dashboard (saved views, metrics, exports)
**Additions:** + Exports & Snapshots
**Spec:** [docs/verticals/2.3-finance-dashboard.md](docs/verticals/2.3-finance-dashboard.md)

**Primitives Delivered:**
- SavedViewStore (OL) - Persist saved dashboard configurations
- DashboardEngine (OL) - Calculate aggregate metrics (sum, count, group by)
- SnapshotExporter (OL) - Generate PDF snapshots of dashboard state
- DashboardGrid (IL) - Responsive grid layout for dashboard widgets
- MetricCard (IL) - Reusable widget for displaying single metric
- SavedViewSelector (IL) - Dropdown for selecting and managing saved views with MRU

**Schemas:** saved-view.schema.json, dashboard-config.schema.json
**UX Flow:** [2.3-finance-dashboard-experience.md](docs/ux-flows/2.3-finance-dashboard-experience.md)

**Delivered:**
- Pre-configured system views (Monthly Overview, Quarterly Summary, Annual Review)
- Custom dashboard creation with configurable widgets
- Saved filter combinations as named views
- Aggregate metrics (income, expenses, net, category breakdown, top merchants, deductibles)
- PDF snapshot export with view configuration metadata
- Drill-down integration to Transaction List View (reuses 2.1)

---

## 🗂️ Group 3: Registries (Finance Domain)

### **3.1 Account**
**Status:** ✅ Complete
**Full Name:** Account Registry (closed registry)
**Spec:** [docs/verticals/3.1-account-registry.md](docs/verticals/3.1-account-registry.md)

**Primitives Delivered:**
- AccountStore (OL) - CRUD operations for user accounts with uniqueness validation
- AccountValidator (OL) - Validates account data (names, types, currencies)
- AccountManager (IL) - Full CRUD UI for managing accounts
- AccountSelector (IL) - Reusable dropdown for selecting account

**Schema:** account.schema.json
**UX Flow:** [3.1-account-registry-experience.md](docs/ux-flows/3.1-account-registry-experience.md)

**Delivered:**
- Closed registry pattern (user manually creates accounts)
- CRUD operations (create, edit, archive/unarchive)
- Unique name enforcement (case-insensitive per user)
- Immutable fields (currency, type cannot change after creation)
- Soft delete (archive preserves transaction history)
- Multi-domain applicability (finance accounts, healthcare providers, legal trust accounts, research funding sources)

---

### **3.2 Counterparty**
**Status:** ✅ Complete
**Status:** ✅ Complete
**Full Name:** Counterparty (open registry)
**Spec:** TBD

**Expected:**
- Open registry = dynamic, grows as new merchants/people appear
- Examples: OpenAI, HubSpot, Uber, Diana de la Tejera
- Attributes: counterparty_id, canonical_name, aliases, type (merchant, person, business)

---

### **3.3 Series**
**Status:** ✅ Complete
**Full Name:** Series Registry (closed registry, recurring payments)
**Spec:** [docs/verticals/3.3-series-registry.md](docs/verticals/3.3-series-registry.md)

**Primitives Delivered:**
- SeriesStore (OL) - CRUD operations for recurring payment series
- RecurrenceEngine (OL) - Calculates next expected date based on frequency patterns
- InstanceTracker (OL) - Auto-links transactions to series instances, detects variances
- SeriesManager (IL) - Full CRUD UI for managing series with status badges
- SeriesSelector (IL) - Dropdown for selecting series to link
- RecurrenceConfigDialog (IL) - Dialog for configuring recurrence patterns

**Schemas:** series.schema.json, series-instance.schema.json
**UX Flow:** [3.3-series-registry-experience.md](docs/ux-flows/3.3-series-registry-experience.md)

**Delivered:**
- Closed registry pattern (user defines recurring payment templates)
- Template + instance tracking (expected vs actual)
- Auto-link transactions to series based on matching criteria (account + counterparty + amount ± tolerance + date ± 3 days)
- Missing payment detection and variance alerts
- Recurrence patterns: daily, weekly, monthly, yearly, custom
- Edge case handling: last day of month (Jan 31 → Feb 28), leap years
- Status badges: ✅ Paid, ⚠️ Variance, 🔴 Missing, 📅 Upcoming
- Multi-domain applicability (finance subscriptions, healthcare premiums, legal retainers, research grants)

---

### **3.4 Tax Categorization**
**Status:** ✅ Complete
**Full Name:** Tax Categorization (Regulatory Classification System)
**Spec:** [docs/verticals/3.4-tax-categorization.md](docs/verticals/3.4-tax-categorization.md)

**Primitives Delivered:**
- TaxCategoryStore (OL) - CRUD for tax categories (system + custom) with hierarchical support
- TaxonomyEngine (OL) - Navigate hierarchical category trees (depth-first, breadth-first)
- TaxCategoryClassifier (OL) - Auto-suggest categories based on patterns (rules + ML)
- FacturaStore (OL) - CRUD for Mexican Factura records (CFDI XML parsing)
- FacturaValidator (OL) - Validate CFDI XML format, RFC, UUID, amount, date
- TaxCategorySelector (IL) - Dropdown for selecting tax category with hierarchy and search
- TaxCategoryManager (IL) - Full UI for managing custom categories (tree view)
- FacturaUploadDialog (IL) - Upload and link Factura XML to transaction

**Schemas:** tax-category.schema.json, tax-classification.schema.json, factura-record.schema.json
**UX Flow:** [3.4-tax-categorization-experience.md](docs/ux-flows/3.4-tax-categorization-experience.md)

**Delivered:**
- Multi-jurisdiction support (USA Schedule C, Mexico SAT, extensible to other jurisdictions)
- Hierarchical taxonomy navigation (parent-child categories)
- Deduction rate tagging (100%, 50%, 0% = compliance levels)
- Factura (CFDI) tracking for Mexico transactions (RFC, UUID validation)
- Auto-classification based on merchant patterns (confidence scoring ≥70%)
- Custom category creation within jurisdictions
- Bulk classification operations
- Multi-domain applicability (Finance → tax, Healthcare → CPT/ICD-10, Legal → case categories, Research → grant compliance)

---

### **3.5 Relationships**
**Status:** ✅ Complete
**Full Name:** Relationships (Transaction-to-Transaction Linking)
**Additions:** + fx_conversion, confidence scoring
**Spec:** [docs/verticals/3.5-relationships.md](docs/verticals/3.5-relationships.md)

**Primitives Delivered:**
- RelationshipStore (OL) - CRUD for transaction relationships (transfer, fx_conversion, reimbursement, split, correction, other)
- TransferDetector (OL) - Auto-detect transfer pairs with confidence scoring (≥0.90 high, 0.70-0.89 medium, 0.50-0.69 low)
- FXConverter (OL) - Calculate exchange rates and FX gain/loss
- RelationshipMatcher (OL) - Fuzzy matching with configurable thresholds
- RelationshipPanel (IL) - Display linked transactions in detail view
- TransferLinkDialog (IL) - Manual link creation UI with search
- FXConversionCard (IL) - FX conversion details display

**Schemas:** relationship.schema.json, relationship-candidate.schema.json, fx-details.schema.json
**UX Flow:** [3.5-relationships-experience.md](docs/ux-flows/3.5-relationships-experience.md)

**Delivered:**
- Auto-detect transfers (same amount, opposite accounts, ±3 days)
- FX conversion tracking with exchange rates and gain/loss
- Manual link creation for edge cases (reimbursements, splits)
- Confidence scoring (0.0-1.0) with weighted features (40% amount, 30% date, 20% signs, 10% accounts)
- Analytics exclusion (prevent double-counting in dashboards)
- Relationship types: transfer, fx_conversion, reimbursement, split, correction, other
- Multi-domain applicability (Finance → transfers, Healthcare → claim-payment, Legal → case relationships, Research → citations, E-commerce → order-return)

**NOTE:** This vertical covers the "Transfer Linking" concept previously mentioned as "1.4"

---

### **3.6 Unit**
**Status:** ✅ Complete
**Full Name:** Unit (Currency & Date Normalization)
**Additions:** + currency/date rules, exchange rate caching
**Spec:** [docs/verticals/3.6-unit.md](docs/verticals/3.6-unit.md)

**Primitives Delivered:**
- CurrencyConverter (OL) - Multi-currency normalization with batch conversion
- DateNormalizer (OL) - Timezone conversion (UTC storage, user display), DST handling
- AmountFormatter (OL) - Locale-aware formatting (en-US, es-MX, de-DE) with currency symbols
- ExchangeRateProvider (OL) - Multi-source rate fetching (ECB, Federal Reserve, manual) with caching
- CurrencySelectorDialog (IL) - Modal dialog for selecting base currency with search
- AmountDisplayCard (IL) - Display original + normalized amounts side-by-side
- ExchangeRateWidget (IL) - Display rate with refresh button and staleness indicator

**Schemas:** normalized-amount.schema.json, exchange-rate-record.schema.json, currency-config.schema.json
**UX Flow:** [3.6-unit-experience.md](docs/ux-flows/3.6-unit-experience.md)

**Delivered:**
- Multi-currency normalization (USD, EUR, MXN, 150+ currencies)
- Original + normalized amount preservation
- Exchange rate fetching with fallback sources (ECB primary, Federal Reserve fallback)
- Rate caching with 24h staleness detection
- Manual rate override with audit trail
- Date normalization (UTC storage, user timezone display)
- ISO 8601 compliance and DST handling
- Amount precision rules (2 decimals for fiat, 4-8 for crypto)
- Locale-aware display formatting (thousands separators, decimal symbols)
- Multi-domain applicability (Finance → multi-currency, Healthcare → international claims, E-commerce → product pricing, Travel → expense reporting, Legal → international billing)

---

### **3.7 Parser Registry**
**Status:** ✅ Complete
**Full Name:** Parser Registry (Service Discovery + Capability-Based Selection)
**Additions:** + capability tracking, auto-selection, versioning
**Spec:** [docs/verticals/3.7-parser-registry.md](docs/verticals/3.7-parser-registry.md)

**Primitives Delivered:**
- ParserRegistry (OL) - CRUD operations for parser registration, discovery, version tracking
- ParserCapabilityStore (OL) - Track extractable fields per parser with confidence scores
- ParserVersionManager (OL) - Version management with deprecation, breaking changes, sunset dates
- ParserSelector (OL) - Auto-select best parser based on file metadata and capabilities
- ParserSelectorDialog (IL) - Modal dialog for manual parser selection with capability badges
- ParserCapabilitiesCard (IL) - Display parser capabilities with confidence indicators
- ParserVersionDropdown (IL) - Version selector with deprecation warnings

**Schemas:** parser-registration.schema.json, parser-capability.schema.json, parser-version.schema.json
**UX Flow:** [3.7-parser-registry-experience.md](docs/ux-flows/3.7-parser-registry-experience.md)

**Delivered:**
- Parser discovery and registration (BofA PDF, Apple Card CSV, Scotia PDF, etc.)
- Capability tracking (which fields each parser extracts: date, amount, merchant, account)
- Semantic versioning with deprecation support (v1.0.0 → v2.1.0)
- Auto-selection based on file type, filename patterns, and required capabilities
- Confidence scoring for parser selection (0.0-1.0)
- Backward compatibility tracking across versions
- Migration guides between breaking versions
- Multi-domain applicability (Finance → PDF/CSV parsers, Healthcare → HL7/FHIR parsers, Legal → document parsers, Research → citation parsers, E-commerce → EDI parsers)

---

### **3.8 Cluster Rules**
**Status:** ✅ Complete
**Full Name:** Cluster Rules
**Spec:** TBD

**Expected:**
- Merchant name normalization rules ("UBER EATS PENDING" → "Uber Eats")
- Clustering similar transactions
- Auto-categorization rules

---

### **3.9 Reconciliation Strategies**
**Status:** ✅ Complete
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
**Status:** ✅ Complete
**Full Name:** Reminders
**Spec:** TBD

**Expected:**
- Alert if recurring payment didn't occur
- Alert if balance below threshold
- Alert if large expense detected
- Due date reminders (credit card payments)

---

### **4.2 Forecast**
**Status:** ✅ Complete
**Full Name:** Forecast
**Spec:** TBD

**Expected:**
- Income/expense projections
- Cash runway calculation
- Goal tracking (monthly income targets)
- Burn rate analysis

---

### **4.3 Corrections Flow**
**Status:** ✅ Complete
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
**Status:** ✅ Complete
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
**Status:** ✅ Complete
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
**Status:** ✅ Complete
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
**Status:** ✅ Complete
**Full Name:** Security & Access (PII, roles)
**Spec:** TBD

**Expected:**
- PII handling (mask account numbers, SSN)
- Role-based access control (owner, accountant read-only)
- Data encryption at rest and in transit
- Audit logs for access

---

### **5.5 Public API Contracts**
**Status:** ✅ Complete
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
| ✅ Complete | 13 | 1.1, 1.2, 1.3, 2.1, 2.2, 2.3, 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7 |
| 📝 Pending | 10 | 3.8-3.9, 4.1-4.3, 5.1-5.5 |
| **TOTAL** | **23** | |

**Completion:** 57% (13/23)

**Next up:** 3.8 Cluster Rules (merchant normalization)

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
