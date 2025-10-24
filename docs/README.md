# Truth Construction Primitives - Finance Instantiation

> **Literate Programming approach:** Multi-domain truth construction primitives demonstrated through finance domain. Each vertical specification mixes prose (WHY), architecture (HOW), and executable contracts (WHAT) in a single source of truth.

---

## 📋 Vertical Specifications

> **📖 [CANONICAL VERTICAL LIST](../VERTICALS.md)** — Single source of truth for all 23 verticals

### Group 1: Upload & Ingestion (Complete ✅)
- **[1.1 Upload Flow](verticals/1.1-upload-flow.md)** - File upload with deduplication and state machine
- **[1.2 Extraction](verticals/1.2-extraction.md)** - Parser execution, raw observation extraction (AS-IS)
- **[1.3 Normalization](verticals/1.3-normalization.md)** - Raw → canonical transformation, validation, categorization

### Group 2: Exploration & Visualization
- **[2.1 Transaction List View](verticals/2.1-transaction-list-view.md)** ✅ Complete - Pagination, filtering, sorting, export
- **[2.2 OL Exploration](verticals/2.2-ol-exploration.md)** ✅ Complete - Drill-down, decisions, provenance, artifact viewing
- **[2.3 Finance Dashboard](verticals/2.3-finance-dashboard.md)** ✅ Complete - Saved views, aggregate metrics, PDF exports

### Group 3: Registries (Finance Domain)
- **[3.1 Account Registry](verticals/3.1-account-registry.md)** ✅ Complete - Closed registry pattern, CRUD operations, soft delete
- **[3.2 Counterparty Registry](verticals/3.2-counterparty-registry.md)** ✅ Complete - Open registry, aliases, fuzzy matching, merge duplicates
- **[3.3 Series Registry](verticals/3.3-series-registry.md)** ✅ Complete - Recurring payments, template + instance tracking, auto-link, variance detection
- **[3.4 Tax Categorization](verticals/3.4-tax-categorization.md)** ✅ Complete - Multi-jurisdiction (USA+Mexico), hierarchical taxonomy, auto-classification, Factura tracking
- **[3.5 Relationships](verticals/3.5-relationships.md)** ✅ Complete - Auto-detect transfers, FX conversion tracking, manual linking, analytics exclusion
- **[3.6 Unit](verticals/3.6-unit.md)** ✅ Complete - Multi-currency normalization, exchange rate caching, date/timezone handling, locale-aware formatting
- **[3.7 Parser Registry](verticals/3.7-parser-registry.md)** ✅ Complete - Parser discovery, capability tracking, versioning, auto-selection

---

## 🧩 Primitives Catalog

### Objective Layer (OL)
**Multi-domain truth construction primitives** (finance instantiation)

These primitives are domain-agnostic - they construct verifiable truth across ANY domain (finance, medicine, legal, etc.). This repository demonstrates their instantiation in the finance domain.

**Vertical 1.1 (Upload Flow):**
- **[StorageEngine](primitives/ol/StorageEngine.md)** - Content-addressable storage with deduplication
- **[ProvenanceLedger](primitives/ol/ProvenanceLedger.md)** - Append-only audit trail with cryptographic integrity
- **[FileArtifact](primitives/ol/FileArtifact.md)** - Internal metadata wrapper for uploaded files
- **[HashCalculator](primitives/ol/HashCalculator.md)** - Streaming SHA-256 hash calculator
- **[UploadRecord](primitives/ol/UploadRecord.md)** - State machine orchestrator for upload flow

**Vertical 1.2 (Extraction):**
- **[Parser](primitives/ol/Parser.md)** - Universal interface for extracting raw observations from documents
- **[ObservationStore](primitives/ol/ObservationStore.md)** - Persistent storage for raw extracted observations (AS-IS)
- **[ParserRegistry](primitives/ol/ParserRegistry.md)** - Service discovery for pluggable parser architecture
- **[ParseLog](primitives/ol/ParseLog.md)** - Structured execution log for parser operations

**Vertical 1.3 (Normalization):**
- **[Normalizer](primitives/ol/Normalizer.md)** - Universal interface for raw → canonical transformation
- **[ValidationEngine](primitives/ol/ValidationEngine.md)** - Rule-based validation for individual fields (dates, amounts, etc.)
- **[CanonicalStore](primitives/ol/CanonicalStore.md)** - Persistent storage for validated canonical records
- **[NormalizationRuleSet](primitives/ol/NormalizationRuleSet.md)** - Externalized, versioned configuration for normalization rules
- **[NormalizationLog](primitives/ol/NormalizationLog.md)** - Structured execution log for normalization operations

**Vertical 2.1 (Transaction List View):**
- **[TransactionQuery](primitives/ol/TransactionQuery.md)** - Universal query builder for canonical stores with filter/sort/pagination
- **[PaginationEngine](primitives/ol/PaginationEngine.md)** - Cursor-based pagination (keyset pagination)
- **[IndexStrategy](primitives/ol/IndexStrategy.md)** - Database index suggestion engine based on query patterns
- **[ExportEngine](primitives/ol/ExportEngine.md)** - Generate CSV/PDF/Excel exports from query results

**Vertical 2.2 (OL Exploration):**
- **[ProvenanceTracer](primitives/ol/ProvenanceTracer.md)** - Trace complete data lineage (canonical → observation → artifact)
- **[DecisionExplainer](primitives/ol/DecisionExplainer.md)** - Explain normalization decisions with rules and confidence
- **[ArtifactRetriever](primitives/ol/ArtifactRetriever.md)** - Retrieve artifacts with signed URLs and access control

**Vertical 2.3 (Finance Dashboard):**
- **[SavedViewStore](primitives/ol/SavedViewStore.md)** - Persist saved dashboard view configurations
- **[DashboardEngine](primitives/ol/DashboardEngine.md)** - Calculate aggregate metrics from canonical data (sum, count, group by)
- **[SnapshotExporter](primitives/ol/SnapshotExporter.md)** - Generate PDF snapshots of dashboard state with metrics and configuration

**Vertical 3.1 (Account Registry):**
- **[AccountStore](primitives/ol/AccountStore.md)** - CRUD operations for user accounts with uniqueness validation and soft delete
- **[AccountValidator](primitives/ol/AccountValidator.md)** - Validates account data (names, types, currencies) before persistence

**Vertical 3.2 (Counterparty Registry):**
- **[CounterpartyStore](primitives/ol/CounterpartyStore.md)** - CRUD operations with find_or_create pattern, alias management, merge support
- **[CounterpartyMatcher](primitives/ol/CounterpartyMatcher.md)** - Fuzzy matching engine (Levenshtein, Jaro-Winkler, token-based)
- **[AliasMerger](primitives/ol/AliasMerger.md)** - Safe merge operations with transaction migration and rollback

**Vertical 3.3 (Series Registry):**
- **[SeriesStore](primitives/ol/SeriesStore.md)** - CRUD operations for recurring payment series with uniqueness validation
- **[RecurrenceEngine](primitives/ol/RecurrenceEngine.md)** - Calculates next expected date based on frequency patterns (daily, weekly, monthly, yearly, custom)
- **[InstanceTracker](primitives/ol/InstanceTracker.md)** - Auto-links transactions to series instances, detects variances and missing payments

**Vertical 3.4 (Tax Categorization):**
- **[TaxCategoryStore](primitives/ol/TaxCategoryStore.md)** - CRUD for tax categories (system + custom) with hierarchical support
- **[TaxonomyEngine](primitives/ol/TaxonomyEngine.md)** - Navigate hierarchical category trees (depth-first, breadth-first)
- **[TaxCategoryClassifier](primitives/ol/TaxCategoryClassifier.md)** - Auto-suggest categories based on patterns (rules + ML)
- **[FacturaStore](primitives/ol/FacturaStore.md)** - CRUD for Mexican Factura records (CFDI XML parsing)
- **[FacturaValidator](primitives/ol/FacturaValidator.md)** - Validate CFDI XML format, RFC, UUID, amount, date

**Vertical 3.5 (Relationships):**
- **[RelationshipStore](primitives/ol/RelationshipStore.md)** - CRUD for transaction relationships (transfer, fx_conversion, reimbursement, split, correction, other)
- **[TransferDetector](primitives/ol/TransferDetector.md)** - Auto-detect transfer pairs with confidence scoring
- **[FXConverter](primitives/ol/FXConverter.md)** - Calculate exchange rates and FX gain/loss
- **[RelationshipMatcher](primitives/ol/RelationshipMatcher.md)** - Fuzzy matching with configurable thresholds

**Vertical 3.6 (Unit):**
- **[CurrencyConverter](primitives/ol/CurrencyConverter.md)** - Multi-currency normalization with batch conversion (150+ currencies)
- **[DateNormalizer](primitives/ol/DateNormalizer.md)** - Timezone conversion (UTC storage, user display) with DST handling and ISO 8601 compliance
- **[AmountFormatter](primitives/ol/AmountFormatter.md)** - Locale-aware formatting (en-US, es-MX, de-DE) with currency symbols and precision rules
- **[ExchangeRateProvider](primitives/ol/ExchangeRateProvider.md)** - Multi-source rate fetching (ECB, Federal Reserve, manual) with caching and staleness detection

**Vertical 3.7 (Parser Registry):**
- **[ParserRegistry](primitives/ol/ParserRegistry.md)** - CRUD operations for parser registration, discovery, version tracking
- **[ParserCapabilityStore](primitives/ol/ParserCapabilityStore.md)** - Track extractable fields per parser with confidence scores
- **[ParserVersionManager](primitives/ol/ParserVersionManager.md)** - Version management with deprecation, breaking changes, sunset dates
- **[ParserSelector](primitives/ol/ParserSelector.md)** - Auto-select best parser based on file metadata and capabilities

### Interface Layer (IL)
Reusable UI components:

- **[FileUpload](primitives/il/FileUpload.md)** - Drag & drop component with validation
- **[TransactionTable](primitives/il/TransactionTable.md)** - Data grid with sorting, pagination, and row selection
- **[DrillDownPanel](primitives/il/DrillDownPanel.md)** - Slide-in panel for complete drill-down view
- **[ProvenanceTimeline](primitives/il/ProvenanceTimeline.md)** - Visual timeline for audit trail
- **[DashboardGrid](primitives/il/DashboardGrid.md)** - Responsive grid layout for dashboard widgets (12-col desktop, stacked mobile)
- **[MetricCard](primitives/il/MetricCard.md)** - Reusable widget for displaying single metric with trend and drill-down
- **[SavedViewSelector](primitives/il/SavedViewSelector.md)** - Dropdown component for selecting and managing saved views with MRU history
- **[AccountManager](primitives/il/AccountManager.md)** - Full CRUD UI for managing user accounts (create, edit, archive, search, filter)
- **[AccountSelector](primitives/il/AccountSelector.md)** - Reusable dropdown for selecting account (used in transaction editing, filtering, reports)
- **[CounterpartyManager](primitives/il/CounterpartyManager.md)** - Full CRUD UI for counterparties with duplicate suggestions and bulk merge
- **[CounterpartySelector](primitives/il/CounterpartySelector.md)** - Dropdown for selecting counterparty with alias search and grouping
- **[MergeCounterpartiesDialog](primitives/il/MergeCounterpartiesDialog.md)** - 5-step wizard for merging duplicate counterparties
- **[SeriesManager](primitives/il/SeriesManager.md)** - Full CRUD UI for managing recurring payment series with status badges and variance alerts
- **[SeriesSelector](primitives/il/SeriesSelector.md)** - Dropdown for selecting series to link (used in transaction detail, manual link dialog)
- **[RecurrenceConfigDialog](primitives/il/RecurrenceConfigDialog.md)** - Dialog for configuring recurrence patterns (daily, weekly, monthly, yearly, custom)
- **[TaxCategorySelector](primitives/il/TaxCategorySelector.md)** - Dropdown for selecting tax category with hierarchy and search
- **[TaxCategoryManager](primitives/il/TaxCategoryManager.md)** - Full UI for managing custom tax categories (tree view)
- **[FacturaUploadDialog](primitives/il/FacturaUploadDialog.md)** - Upload and link Factura XML to transaction (Mexico CFDI)
- **[RelationshipPanel](primitives/il/RelationshipPanel.md)** - Display linked transactions in detail view
- **[TransferLinkDialog](primitives/il/TransferLinkDialog.md)** - Manual link creation UI with transaction search
- **[FXConversionCard](primitives/il/FXConversionCard.md)** - FX conversion details display with market rate comparison
- **[CurrencySelectorDialog](primitives/il/CurrencySelectorDialog.md)** - Modal dialog for selecting base currency with search and popular currencies
- **[AmountDisplayCard](primitives/il/AmountDisplayCard.md)** - Display original + normalized amounts side-by-side with toggle
- **[ExchangeRateWidget](primitives/il/ExchangeRateWidget.md)** - Display rate with refresh button, staleness indicator, and source badge
- **[ParserSelectorDialog](primitives/il/ParserSelectorDialog.md)** - Modal dialog for manual parser selection with capability badges
- **[ParserCapabilitiesCard](primitives/il/ParserCapabilitiesCard.md)** - Display parser capabilities with confidence indicators
- **[ParserVersionDropdown](primitives/il/ParserVersionDropdown.md)** - Version selector with deprecation warnings
- **[IL Components Summary](primitives/il/_IL_COMPONENTS_SUMMARY.md)** - Catalog of all IL components

---

## 📐 JSON Schemas

Executable contracts extracted from vertical specifications:

**Vertical 1.1 (Upload):**
- **[upload-record.schema.json](schemas/upload-record.schema.json)** - UploadRecord state machine contract
- **[provenance-entry.schema.json](schemas/provenance-entry.schema.json)** - ProvenanceLedger entry format
- **[upload-error-response.schema.json](schemas/upload-error-response.schema.json)** - Standardized error responses

**Vertical 1.2 (Extraction):**
- **[observation-transaction.schema.json](schemas/observation-transaction.schema.json)** - Raw extracted observations (AS-IS from parser)
- **[parse-log.schema.json](schemas/parse-log.schema.json)** - Parser execution log

**Vertical 1.3 (Normalization):**
- **[canonical-transaction.schema.json](schemas/canonical-transaction.schema.json)** - Validated and normalized canonical transactions
- **[normalization-log.schema.json](schemas/normalization-log.schema.json)** - Normalization execution log with validation results

**Vertical 2.1 (Transaction List View):**
- **[transaction-query-response.schema.json](schemas/transaction-query-response.schema.json)** - API response format for paginated query results

**Vertical 2.2 (OL Exploration):**
- **[drill-down-response.schema.json](schemas/drill-down-response.schema.json)** - Complete data lineage response (canonical + observation + artifact + decisions + provenance)
- **[decision-explanation.schema.json](schemas/decision-explanation.schema.json)** - Normalization decision explanation format

**Vertical 2.3 (Finance Dashboard):**
- **[saved-view.schema.json](schemas/saved-view.schema.json)** - Saved dashboard view configuration (system + user views)
- **[dashboard-config.schema.json](schemas/dashboard-config.schema.json)** - Dashboard widget layout, filters, and date range configuration

**Vertical 3.1 (Account Registry):**
- **[account.schema.json](schemas/account.schema.json)** - User account entity (name, type, currency, institution, active status)

**Vertical 3.2 (Counterparty Registry):**
- **[counterparty.schema.json](schemas/counterparty.schema.json)** - Counterparty entity with canonical name, aliases array, merge support

**Vertical 3.3 (Series Registry):**
- **[series.schema.json](schemas/series.schema.json)** - Recurring payment series definition (expected amount, frequency pattern, tolerance)
- **[series-instance.schema.json](schemas/series-instance.schema.json)** - Instance tracking record (expected vs actual, variance, status)

**Vertical 3.4 (Tax Categorization):**
- **[tax-category.schema.json](schemas/tax-category.schema.json)** - Tax category definition (multi-jurisdiction, hierarchical, deduction rate)
- **[tax-classification.schema.json](schemas/tax-classification.schema.json)** - Transaction tax classification (category, deduction rate, confidence)
- **[factura-record.schema.json](schemas/factura-record.schema.json)** - Mexican Factura (CFDI) record (RFC, UUID, amount, date)

**Vertical 3.5 (Relationships):**
- **[relationship.schema.json](schemas/relationship.schema.json)** - Transaction relationship entity (transfer, fx_conversion, reimbursement, split, correction, other)
- **[relationship-candidate.schema.json](schemas/relationship-candidate.schema.json)** - Auto-detection result with confidence score and match details
- **[fx-details.schema.json](schemas/fx-details.schema.json)** - FX conversion details (exchange rate, market rate, gain/loss)

**Vertical 3.6 (Unit):**
- **[normalized-amount.schema.json](schemas/normalized-amount.schema.json)** - Amount with original + normalized values and exchange rate metadata
- **[exchange-rate-record.schema.json](schemas/exchange-rate-record.schema.json)** - Cached exchange rate record with staleness detection
- **[currency-config.schema.json](schemas/currency-config.schema.json)** - User currency configuration (base currency, timezone, display locale, precision)

**Vertical 3.7 (Parser Registry):**
- **[parser-registration.schema.json](schemas/parser-registration.schema.json)** - Parser registration record with metadata, version, and capabilities
- **[parser-capability.schema.json](schemas/parser-capability.schema.json)** - Capability definition for a parser (extractable fields with confidence)
- **[parser-version.schema.json](schemas/parser-version.schema.json)** - Parser version record with deprecation and compatibility info

---

## 🏛️ Architecture Decision Records (ADR)

Key architectural decisions with rationale:

- **[ADR-0001: Canonical ID Decision](adr/0001-canonical-id-decision.md)** - Why `upload_id` not `file_id`
- **[ADR-0002: State Machine Global](adr/0002-state-machine-global.md)** - Why single `status` field
- **[ADR-0003: Runner/Coordinator Split](adr/0003-runner-coordinator-split.md)** - Separation of concerns
- **[ADR-0004: Raw-First Extraction](adr/0004-raw-first-extraction.md)** - Store AS-IS, validate later
- **[ADR-0005: Configurable Normalization Rules](adr/0005-configurable-normalization-rules.md)** - Externalized, versioned rule sets

---

## 🎨 UX Flows

User experience specifications with wireframes and journeys:

- **[1.1 Upload Experience](ux-flows/1.1-upload-experience.md)** - File upload UX, drag & drop, validation feedback
- **[1.2 Extraction Experience](ux-flows/1.2-extraction-experience.md)** - Parsing status, error handling, retry flows
- **[1.3 Normalization Experience](ux-flows/1.3-normalization-experience.md)** - Validation results, error review, duplicate resolution, categorization
- **[2.1 Transaction List Experience](ux-flows/2.1-transaction-list-experience.md)** - Filtering, sorting, pagination, export workflows
- **[2.2 OL Exploration Experience](ux-flows/2.2-ol-exploration-experience.md)** - Drill-down, decision explanations, provenance timeline, artifact viewing
- **[2.3 Finance Dashboard Experience](ux-flows/2.3-finance-dashboard-experience.md)** - Dashboard views, custom view creation, metric drill-down, PDF export
- **[3.1 Account Registry Experience](ux-flows/3.1-account-registry-experience.md)** - Account creation, editing, archiving, selector dropdown, immutability patterns
- **[3.2 Counterparty Registry Experience](ux-flows/3.2-counterparty-registry-experience.md)** - Auto-creation, fuzzy matching, merge duplicates, alias management
- **[3.3 Series Registry Experience](ux-flows/3.3-series-registry-experience.md)** - Create recurring series, view status badges, manual link, variance alerts, archive series
- **[3.4 Tax Categorization Experience](ux-flows/3.4-tax-categorization-experience.md)** - Classify transactions, upload Factura, create custom categories, auto-suggestions, bulk operations
- **[3.5 Relationships Experience](ux-flows/3.5-relationships-experience.md)** - Accept transfer suggestion, manual link creation, FX conversion viewing, unlink flow, multiple candidates
- **[3.6 Unit Experience](ux-flows/3.6-unit-experience.md)** - Set base currency, view dual currencies, refresh stale rates, manual rate override, timezone change
- **[3.7 Parser Registry Experience](ux-flows/3.7-parser-registry-experience.md)** - Auto-detected parser, manual override, view capabilities, deprecated parser warning, register new parser

---

## 📊 Progress

| Group | Vertical | Status |
|-------|----------|--------|
| **1. Upload & Ingestion** | 1.1 Upload Flow | ✅ Complete |
| | 1.2 Extraction | ✅ Complete |
| | 1.3 Normalization | ✅ Complete |
| **2. Exploration & Viz** | 2.1 Transaction List View | ✅ Complete |
| | 2.2 OL Exploration | ✅ Complete |
| | 2.3 Finance Dashboard | ✅ Complete |
| **3. Registries** | 3.1 Account Registry | ✅ Complete |
| | 3.2 Counterparty Registry | ✅ Complete |
| | 3.3 Series Registry | ✅ Complete |
| | 3.4 Tax Categorization | ✅ Complete |
| | 3.5 Relationships | ✅ Complete |
| | 3.6 Unit | ✅ Complete |
| | 3.7 Parser Registry | ✅ Complete |
| | 3.8-3.9 | 📝 Pending |
| **4. Derivatives** | 4.1-4.3 | 📝 Pending |
| **5. Governance** | 5.1-5.5 | 📝 Pending |

---

## 🏗️ Methodology

Each vertical follows a **20-section checklist:**

### Product Layer (Sections 1-10)
What the user sees and experiences

### Machinery Layer (Sections 11-15)
How primitives power the vertical

### Cross-Cutting Concerns (Sections 16-20)
Security, performance, observability, testing, operations

---

## 📦 Repository Philosophy

This repository contains **only specification artifacts** - the crystallized output of vertical analysis.

**What's included:**
- ✅ Vertical specifications (`.md`)
- ✅ JSON schemas (`.schema.json`)
- ✅ Primitive specs (OL/IL `.md`)
- ✅ ADRs (Architecture Decision Records)
- ✅ UX flows and wireframes

**What's excluded:**
- ❌ Implementation code (lives elsewhere)
- ❌ Internal coordination files
- ❌ Status reports

---

## 🚀 Navigation

- **For Product/Design:** Start with [verticals/](verticals/) and [ux-flows/](ux-flows/)
- **For Engineering:** Read [verticals/](verticals/) → [primitives/](primitives/) → [schemas/](schemas/)
- **For Architecture:** Focus on [adr/](adr/) and [primitives/ol/](primitives/ol/)

---

*This documentation grows incrementally as each vertical is analyzed and crystallized.*
