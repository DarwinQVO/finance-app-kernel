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
- **[3.8 Cluster Rules](verticals/3.8-cluster-rules.md)** ✅ Complete - Merchant name normalization, fuzzy matching, transaction clustering, rule engine
- **[3.9 Reconciliation Strategies](verticals/3.9-reconciliation-strategies.md)** ✅ Complete - Multi-source matching, confidence scoring, one-to-many reconciliation, blocking optimization

### Group 4: Derivatives & Insights
- **[4.1 Reminders](verticals/4.1-reminders.md)** ✅ Complete - Alert rules, notification delivery, snooze/dismiss, persistence, multi-channel dispatch
- **[4.2 Forecast](verticals/4.2-forecast.md)** ✅ Complete - Income/expense projections, goal tracking, burn rate, cash runway, scenario analysis
- **[4.3 Corrections Flow](verticals/4.3-corrections-flow.md)** ✅ Complete - Field-level overrides, audit trail, precedence resolution, validation engine, bulk corrections, revert capability

### Group 5: Governance & Meta
- **[5.1 Provenance Ledger](verticals/5.1-provenance-ledger.md)** ✅ Complete - Bitemporal tracking (transaction time + valid time), "as of" queries, retroactive corrections, timeline visualization, immutable audit trail, compliance (HIPAA/SOX/GDPR)

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

**Vertical 3.8 (Cluster Rules):**
- **[MerchantNormalizer](primitives/ol/MerchantNormalizer.md)** - Apply normalization rules to merchant names with batch processing
- **[ClusteringEngine](primitives/ol/ClusteringEngine.md)** - Create clusters of similar transactions using threshold-based grouping
- **[NormalizationRuleStore](primitives/ol/NormalizationRuleStore.md)** - CRUD operations for user-defined normalization rules with precedence
- **[FuzzyMatcher](primitives/ol/FuzzyMatcher.md)** - Text similarity calculation with multiple algorithms (Levenshtein, Jaro-Winkler, Soundex, Metaphone)

**Vertical 3.9 (Reconciliation Strategies):**
- **[ReconciliationEngine](primitives/ol/ReconciliationEngine.md)** - Core orchestrator with blocking strategies, find_candidates(), bulk_reconcile(), multi-cardinality support
- **[MatchScorer](primitives/ol/MatchScorer.md)** - Weighted similarity scoring (40% amount, 30% date, 20% counterparty, 10% description) with fuzzy matching
- **[ThresholdManager](primitives/ol/ThresholdManager.md)** - Decision rule engine, threshold management (auto-link ≥0.95, suggest 0.70-0.94, manual 0.50-0.69)
- **[ReconciliationStore](primitives/ol/ReconciliationStore.md)** - CRUD for reconciliation results with audit trail, orphaned item handling, cascading deletes

**Vertical 4.1 (Reminders):**
- **[ReminderEngine](primitives/ol/ReminderEngine.md)** - Core orchestrator for alert rule evaluation, scheduling, and notification triggering
- **[AlertRuleEvaluator](primitives/ol/AlertRuleEvaluator.md)** - Evaluates user-defined conditions against canonical data (thresholds, patterns, anomalies)
- **[NotificationDispatcher](primitives/ol/NotificationDispatcher.md)** - Multi-channel notification delivery (in-app, email, SMS, push) with retry logic
- **[ReminderStore](primitives/ol/ReminderStore.md)** - CRUD for alert rules, notification history, snooze state, delivery receipts

**Vertical 4.2 (Forecast):**
- **[ProjectionEngine](primitives/ol/ProjectionEngine.md)** - Forecasting algorithms (Holt-Winters, linear regression, moving average, ARIMA, Prophet) with accuracy metrics
- **[GoalStore](primitives/ol/GoalStore.md)** - CRUD for financial goals with progress tracking, milestone alerts, and recurring contributions
- **[BurnRateCalculator](primitives/ol/BurnRateCalculator.md)** - Calculate burn rate (simple, weighted, EMA), cash runway, zero date prediction
- **[ForecastCache](primitives/ol/ForecastCache.md)** - Cache projections with 24h TTL, smart invalidation on data changes

**Vertical 4.3 (Corrections Flow):**
- **[OverrideStore](primitives/ol/OverrideStore.md)** - CRUD for field-level overrides with bulk operations support (1,000 items in <5s)
- **[AuditLog](primitives/ol/AuditLog.md)** - Immutable audit trail logging for all field changes (extraction, override, revert, delete)
- **[PrecedenceEngine](primitives/ol/PrecedenceEngine.md)** - Resolve field values using precedence rules (manual > rule > extraction > default)
- **[ValidationEngine](primitives/ol/ValidationEngine.md)** - Validate field overrides before accepting (type, range, format, business logic validation)

**Vertical 5.1 (Provenance Ledger):**
- **[ProvenanceLedger](primitives/ol/ProvenanceLedger.md)** - Bitemporal event storage (append-only, SHA-256 signatures, <100ms queries on 10M+ records)
- **[BitemporalQuery](primitives/ol/BitemporalQuery.md)** - Query provenance with transaction time and/or valid time filters ("as of" queries)
- **[TimelineReconstructor](primitives/ol/TimelineReconstructor.md)** - Reconstruct complete entity history across both timelines for visualization
- **[RetroactiveCorrector](primitives/ol/RetroactiveCorrector.md)** - Handle retroactive corrections with validation and impact analysis

**Vertical 5.2 (Schema Registry):**
- **[SchemaRegistry](primitives/ol/SchemaRegistry.md)** - Schema version storage and retrieval with UNIQUE(schema_id, version) constraint, <3ms p50 queries
- **[SchemaVersionManager](primitives/ol/SchemaVersionManager.md)** - Semantic version management (MAJOR.MINOR.PATCH), lifecycle operations (publish, deprecate, list versions)
- **[MigrationEngine](primitives/ol/MigrationEngine.md)** - Zero-downtime migrations (shadow tables, batch processing 14K records/sec, automatic rollback on failure)
- **[BackwardCompatibilityChecker](primitives/ol/BackwardCompatibilityChecker.md)** - Detect breaking changes with field-level JSON Schema diff, 15+ change types (field removed, type changed, etc.), <380ms p95 for 100-field schemas

**Vertical 5.3 (Rule Performance & Logs):**
- **[MetricsCollector](primitives/ol/MetricsCollector.md)** - Record execution metrics for parsers and rules with <10ms p95 ingestion, batch insert support (10K metrics/sec)
- **[PerformanceAnalyzer](primitives/ol/PerformanceAnalyzer.md)** - Query and aggregate metrics (latency percentiles, success rates, throughput), <50ms p95 single parser stats, <200ms all parsers
- **[QueueMonitor](primitives/ol/QueueMonitor.md)** - Monitor queue depths and detect backlog/stuck documents, <20ms p95 current depth query, snapshot every 60 seconds
- **[TrendAnalyzer](primitives/ol/TrendAnalyzer.md)** - Analyze historical trends and detect performance degradation, anomaly detection (z-score >2.5), capacity forecasting, <150ms p95 trend queries

**Vertical 5.4 (Security & Access):**
- **[PIIMasker](primitives/ol/PIIMasker.md)** - Detect and mask PII (SSN, credit card, email, phone, IP) using pattern-based + semantic rules, <5ms p95 per observation, 3 strategies (partial/full/tokenize)
- **[AccessControl](primitives/ol/AccessControl.md)** - RBAC permission checks with 5 roles (owner/editor/viewer/accountant/auditor), PostgreSQL RLS integration, <2ms p95 cached checks
- **[EncryptionEngine](primitives/ol/EncryptionEngine.md)** - AES-256-GCM encryption with envelope encryption (Master Key → DEK → Data), AWS KMS integration, 90-day key rotation, <10ms p95 for 5MB docs
- **[AuditLogger](primitives/ol/AuditLogger.md)** - Immutable audit trail with SHA-256 integrity chain, 7-year retention, compliance exports (JSON/CSV/PDF), <8ms p95 async writes

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
- **[MerchantRulesManager](primitives/il/MerchantRulesManager.md)** - Full CRUD UI for managing normalization rules with priority
- **[ClusterViewer](primitives/il/ClusterViewer.md)** - Display transaction clusters with expand/collapse and manual adjustments
- **[RuleEditorDialog](primitives/il/RuleEditorDialog.md)** - Create/edit rule dialog with pattern testing and preview
- **[ReconciliationDashboard](primitives/il/ReconciliationDashboard.md)** - Three-column UI showing unmatched items, suggested matches, matched items with progress tracking
- **[MatchReviewDialog](primitives/il/MatchReviewDialog.md)** - Side-by-side comparison dialog with confidence breakdown, accept/reject actions
- **[ManualMatchDialog](primitives/il/ManualMatchDialog.md)** - Manual match creation with search, multi-select for one-to-many, amount validation
- **[ReminderBell](primitives/il/ReminderBell.md)** - Bell icon with badge counter, dropdown for recent notifications, unread indicator
- **[NotificationPanel](primitives/il/NotificationPanel.md)** - Full notification center with filtering, mark as read, snooze/dismiss actions
- **[ReminderConfigDialog](primitives/il/ReminderConfigDialog.md)** - Create/edit alert rules with condition builder, channel selection, scheduling
- **[ForecastChart](primitives/il/ForecastChart.md)** - Interactive line/area/bar chart for historical data + projections with confidence intervals, algorithm comparison
- **[GoalProgressCard](primitives/il/GoalProgressCard.md)** - Goal display with progress bar, on-track status, time/amount remaining, milestone tracking
- **[GoalConfigDialog](primitives/il/GoalConfigDialog.md)** - Modal for creating/editing goals with templates, tabbed interface (Basic/Advanced/Preview), live preview
- **[CorrectionDialog](primitives/il/CorrectionDialog.md)** - Modal/drawer for editing field values with inline validation, single-item and bulk modes, field history viewer
- **[AuditTrailViewer](primitives/il/AuditTrailViewer.md)** - Timeline visualization showing complete history of field changes with filters, export capability
- **[FieldOverrideIndicator](primitives/il/FieldOverrideIndicator.md)** - Badge/icon showing field was manually corrected with tooltip (who, when), clickable to open audit trail
- **[TimelineViewer](primitives/il/TimelineViewer.md)** - Bitemporal timeline visualization (D3.js) with dual axes (transaction time, valid time), event color coding, zoom/pan, export
- **[AsOfQueryBuilder](primitives/il/AsOfQueryBuilder.md)** - Date picker interface for building "as of" queries (transaction time, valid time, bitemporal modes)
- **[RetroactiveCorrectionDialog](primitives/il/RetroactiveCorrectionDialog.md)** - Modal for making retroactive corrections with effective date selector (Today, Original Date, Custom)
- **[SchemaEditor](primitives/il/SchemaEditor.md)** - Monaco-based JSON Schema editor with autocomplete, syntax highlighting, real-time validation, version comparison
- **[MigrationWizard](primitives/il/MigrationWizard.md)** - 5-step wizard for schema migrations (pre-checks, backup, shadow setup, transformation, cutover) with real-time progress
- **[CompatibilityViewer](primitives/il/CompatibilityViewer.md)** - Side-by-side diff viewer for schema versions with breaking change detection, field-level comparison, export
- **[PerformanceDashboard](primitives/il/PerformanceDashboard.md)** - Real-time monitoring dashboard (4 panels: parser stats, rule stats, queue health, error breakdown), auto-refresh 60s, drill-down navigation
- **[RuleOptimizer](primitives/il/RuleOptimizer.md)** - Diagnostic tool for identifying slow normalization rules, AI-powered optimization recommendations, A/B testing sandbox, impact prediction
- **[QueueMonitorPanel](primitives/il/QueueMonitorPanel.md)** - Real-time queue depth monitoring (pending/in-progress/stuck), health badges (healthy/warning/critical), manual controls (pause/resume/retry)
- **[SecurityDashboard](primitives/il/SecurityDashboard.md)** - Real-time security overview (4 panels: access denials chart, PII activity table, security events timeline, user activity heatmap), auto-refresh 60s, time range 24h/7d/30d
- **[RoleManager](primitives/il/RoleManager.md)** - User role assignment UI with role hierarchy visualization, permission preview, bulk assignment, temporary roles with expiration
- **[AuditViewer](primitives/il/AuditViewer.md)** - Paginated audit log browser with filters (date range, user, event type, result), CSV/PDF export, TanStack Table virtualization (1M+ events)
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

**Vertical 3.8 (Cluster Rules):**
- **[normalization-rule.schema.json](schemas/normalization-rule.schema.json)** - User-defined normalization rule (pattern, replacement, type, priority)
- **[merchant-cluster.schema.json](schemas/merchant-cluster.schema.json)** - Transaction cluster entity with normalized name, member transactions, confidence
- **[rule-execution-log.schema.json](schemas/rule-execution-log.schema.json)** - Rule execution log with match details and performance metrics

**Vertical 3.9 (Reconciliation Strategies):**
- **[reconciliation-result.schema.json](schemas/reconciliation-result.schema.json)** - Reconciliation result with matched pairs, confidence, method (auto|manual), cardinality (one-to-one|one-to-many|many-to-one)
- **[match-candidate.schema.json](schemas/match-candidate.schema.json)** - Match candidate with similarity scores, feature breakdown (amount, date, counterparty, description), decision
- **[reconciliation-config.schema.json](schemas/reconciliation-config.schema.json)** - Configuration for thresholds, tolerances, weights, blocking parameters, performance settings

**Vertical 4.1 (Reminders):**
- **[reminder-config.schema.json](schemas/reminder-config.schema.json)** - Alert rule configuration (conditions, triggers, channels, schedule, snooze settings)
- **[notification-event.schema.json](schemas/notification-event.schema.json)** - Notification event record (type, payload, delivery status, timestamp, read state)
- **[alert-condition.schema.json](schemas/alert-condition.schema.json)** - Alert condition definition (field, operator, threshold, pattern, anomaly detection parameters)

**Vertical 4.2 (Forecast):**
- **[forecast-projection.schema.json](schemas/forecast-projection.schema.json)** - Projection result with historical data, forecasted values, confidence intervals, algorithm metadata
- **[goal-config.schema.json](schemas/goal-config.schema.json)** - Goal configuration (target amount, deadline, category, recurring contributions, milestones)
- **[burn-rate-analysis.schema.json](schemas/burn-rate-analysis.schema.json)** - Burn rate calculation result (daily/weekly/monthly rates, runway months, zero date prediction)

**Vertical 4.3 (Corrections Flow):**
- **[field-override.schema.json](schemas/field-override.schema.json)** - Field-level override with original value, corrected value, user metadata, reason, approval workflow
- **[audit-entry.schema.json](schemas/audit-entry.schema.json)** - Immutable audit log entry (action type, before/after values, user, timestamp, source, signature for tamper detection)
- **[precedence-resolution.schema.json](schemas/precedence-resolution.schema.json)** - Final field value after precedence resolution (manual > rule > extraction > default) with all sources visible

**Vertical 5.1 (Provenance Ledger):**
- **[provenance-record.schema.json](schemas/provenance-record.schema.json)** - Bitemporal provenance record (transaction_time, valid_time_start/end, value, signature for tamper detection)
- **[bitemporal-query.schema.json](schemas/bitemporal-query.schema.json)** - Query structure for "as of" queries (transaction time, valid time, bitemporal modes, filters, pagination)
- **[timeline-event.schema.json](schemas/timeline-event.schema.json)** - Timeline event for visualization (transaction_time, valid_time, action, value, metadata for color/icon)

**Vertical 5.2 (Schema Registry):**
- **[schema-version.schema.json](schemas/schema-version.schema.json)** - Versioned schema definition (schema_id, version, JSON Schema, backward_compatible flag, breaking/additive changes, changelog, deprecation, migration metadata)
- **[migration-plan.schema.json](schemas/migration-plan.schema.json)** - Migration execution plan (from/to versions, steps, batch size, rollback strategy, estimated duration, pre-migration validation)
- **[compatibility-report.schema.json](schemas/compatibility-report.schema.json)** - Compatibility analysis result (compatible flag, breaking changes with severity, impact analysis, recommended version bump)

**Vertical 5.3 (Rule Performance & Logs):**
- **[parser-execution-metric.schema.json](schemas/parser-execution-metric.schema.json)** - Parser execution metric record (parser_id, duration_ms, status, observations_extracted, error_type, document_size_bytes)
- **[rule-execution-metric.schema.json](schemas/rule-execution-metric.schema.json)** - Rule execution metric record (rule_id, rule_type, execution_time_ms, matched, transformation_applied, confidence_score)
- **[queue-depth-snapshot.schema.json](schemas/queue-depth-snapshot.schema.json)** - Queue depth snapshot (queue_name, pending, in_progress, stuck, health_status, worker_stats, alerts)

**Vertical 5.4 (Security & Access):**
- **[pii-mask-rule.schema.json](schemas/pii-mask-rule.schema.json)** - PII masking rule configuration (field_pattern, pii_type, strategy, show_last_digits, exceptions, compliance_tags)
- **[access-policy.schema.json](schemas/access-policy.schema.json)** - RBAC access policy (user_id, role, resource_type, resource_id, actions, conditions, expiration, revocation)
- **[audit-event.schema.json](schemas/audit-event.schema.json)** - Audit trail event (event_type, user_id, action, resource_type, result, integrity_hash, compliance_tags, 7-year retention)

---

## 🏛️ Architecture Decision Records (ADR)

Key architectural decisions with rationale:

**Group 1: Upload & Ingestion**
- **[ADR-0001: Canonical ID Decision](adr/0001-canonical-id-decision.md)** - Why `upload_id` not `file_id`
- **[ADR-0002: State Machine Global](adr/0002-state-machine-global.md)** - Why single `status` field
- **[ADR-0003: Runner/Coordinator Split](adr/0003-runner-coordinator-split.md)** - Separation of concerns
- **[ADR-0004: Raw-First Extraction](adr/0004-raw-first-extraction.md)** - Store AS-IS, validate later
- **[ADR-0005: Configurable Normalization Rules](adr/0005-configurable-normalization-rules.md)** - Externalized, versioned rule sets

**Group 2: Exploration & Visualization**
- **[ADR-0006: Cursor-Based Pagination Strategy](adr/0006-cursor-based-pagination.md)** - Keyset pagination for stability under concurrent inserts
- **[ADR-0007: Provenance Traceability Architecture](adr/0007-provenance-traceability.md)** - Full trace canonical → observation → artifact with signed URLs
- **[ADR-0008: Saved Views Persistence Strategy](adr/0008-saved-views-persistence.md)** - JSON config in database for cross-device sync

**Group 3: Registries**
- **[ADR-0009: Counterparty Merge Strategy](adr/0009-counterparty-merge-strategy.md)** - Cascade update with transaction migration
- **[ADR-0010: Auto-Link Matching Criteria](adr/0010-auto-link-matching-criteria.md)** - Exact account/counterparty + ±5% amount + ±3 day tolerance
- **[ADR-0011: Multi-Jurisdiction Taxonomy Design](adr/0011-multi-jurisdiction-taxonomy.md)** - Hierarchical tree per jurisdiction (USA, Mexico)
- **[ADR-0012: Transfer Detection Confidence Scoring](adr/0012-transfer-detection-scoring.md)** - Weighted multi-feature (40% amount, 30% date, 20% signs, 10% accounts)
- **[ADR-0013: Exchange Rate Caching Strategy](adr/0013-exchange-rate-caching.md)** - 24-hour cache with staleness detection and fallback sources
- **[ADR-0014: Parser Auto-Selection Algorithm](adr/0014-parser-auto-selection.md)** - Capability-based selection with confidence scoring
- **[ADR-0015: Rule Precedence System](adr/0015-rule-precedence.md)** - Priority-based (0-100) with type fallback (exact > regex > fuzzy > soundex)
- **[ADR-0016: Reconciliation Threshold Strategy](adr/0016-reconciliation-thresholds.md)** - 3-tier thresholds (auto-link ≥0.95, suggest 0.70-0.94, manual 0.50-0.69)
- **[ADR-0017: Blocking Strategy for Performance](adr/0017-blocking-strategy-performance.md)** - Date ±30d and amount ±20% pre-filtering for 150x speedup

**Group 4: Derivatives & Insights**
- **[ADR-0018: Notification Delivery Strategy](adr/0018-notification-delivery-strategy.md)** - Multi-channel dispatch with priority queue, retry logic, and delivery receipts
- **[ADR-0019: Alert Rule Engine Architecture](adr/0019-alert-rule-engine-architecture.md)** - Event-driven evaluation with scheduled polling, threshold-based triggers, and condition composition
- **[ADR-0020: Reminder Snooze & Persistence Strategy](adr/0020-reminder-snooze-persistence.md)** - User-specific snooze state with exponential backoff and cross-device synchronization
- **[ADR-0021: Projection Algorithm Strategy](adr/0021-projection-algorithm-strategy.md)** - Multi-algorithm ensemble with accuracy tracking, algorithm selection based on data characteristics
- **[ADR-0022: Goal Tracking Persistence Strategy](adr/0022-goal-tracking-persistence.md)** - Milestone-based tracking with auto-progression, recurring contribution handling, achievement notifications
- **[ADR-0023: Forecast Caching Strategy](adr/0023-forecast-caching-strategy.md)** - 24-hour projection cache with smart invalidation on transaction changes, pre-warming for common queries
- **[ADR-0024: Field-Level Overrides](adr/0024-field-level-overrides.md)** - Field-level granularity (vs record-level) for corrections with independent override capability per field
- **[ADR-0025: Audit Storage Strategy](adr/0025-audit-storage-strategy.md)** - PostgreSQL append-only table with JSONB metadata, database-enforced immutability, cryptographic signatures
- **[ADR-0026: Precedence Resolution](adr/0026-precedence-resolution.md)** - Static precedence rules (manual > rule > extraction > default), predictable and deterministic value resolution
- **[ADR-0027: Bitemporal Model](adr/0027-bitemporal-model.md)** - Full bitemporal tracking (transaction time + valid time) for retroactive corrections and audit compliance
- **[ADR-0028: Provenance Storage Strategy](adr/0028-provenance-storage-strategy.md)** - PostgreSQL append-only with monthly partitioning, S3 archival for >2 years, <100ms queries on 10M+ records
- **[ADR-0029: Query Performance Optimization](adr/0029-query-performance-optimization.md)** - Layered optimization (indexes + materialized views + Redis cache + partitioning) for sub-100ms "as of" queries

**Group 5: Governance & Meta**
- **[ADR-0030: Semantic Versioning Strategy](adr/0030-versioning-strategy.md)** - MAJOR.MINOR.PATCH versioning for schemas with strict compatibility rules, rejected alternatives (timestamp versioning, auto-increment, git SHA), <3ms p50 queries
- **[ADR-0031: Migration Execution - Batch Processing](adr/0031-migration-execution.md)** - Shadow table strategy with batch processing (10K records/batch), zero-downtime cutover (2.5s), automatic rollback on failure, 14,183 records/sec throughput
- **[ADR-0032: Breaking Change Detection Algorithm](adr/0032-breaking-change-detection.md)** - Field-level JSON Schema diff with semantic analysis, 15+ breaking change types detected (field removed, type changed, constraint tightened), <380ms p95 for 100-field schemas
- **[ADR-0033: Metrics Storage Strategy](adr/0033-metrics-storage-strategy.md)** - PostgreSQL with TimescaleDB extension for time-series optimization, 10K metrics/sec batch insert, monthly partitions, <50ms p95 aggregate queries, 90-day hot retention + S3 archive
- **[ADR-0034: Aggregation Performance Strategy](adr/0034-aggregation-performance.md)** - Two-tier caching (TimescaleDB continuous aggregates + Redis cache), 22x faster queries (8ms vs 180ms), 96% cache hit rate, 5-minute refresh, 60-second TTL
- **[ADR-0035: Alerting Architecture](adr/0035-alerting-architecture.md)** - Poll-based alerting with 60-second evaluation interval, <30s average alert latency, stateful tracking, fingerprint-based deduplication, no write path impact
- **[ADR-0036: PII Masking Strategy](adr/0036-pii-masking-strategy.md)** - Rule-based + semantic detection with partial masking and PostgreSQL token vault, 98% accuracy, <5ms p95, 200 obs/sec throughput, alternatives rejected: ML-based (45ms), AWS Macie ($51K/month)
- **[ADR-0037: RBAC Implementation](adr/0037-rbac-implementation.md)** - Hierarchical RBAC (5 roles) + PostgreSQL RLS + Redis cache, <2ms p95 checks (98% hit rate), defense-in-depth, alternatives rejected: ABAC/OPA (8.5ms), OAuth2 JWT (no revocation)
- **[ADR-0038: Encryption Approach](adr/0038-encryption-approach.md)** - Envelope encryption (AES-256-GCM) + AWS KMS, <10ms p95 encrypt/decrypt 5MB docs, quarterly key rotation <30s cutover, $130/month, alternatives rejected: Direct KMS ($30K/month), dedicated HSM ($10K/month)

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
- **[3.8 Cluster Rules Experience](ux-flows/3.8-cluster-rules-experience.md)** - Create normalization rule, view clusters, test rule pattern, manual cluster adjustments, bulk apply rules
- **[3.9 Reconciliation Strategies Experience](ux-flows/3.9-reconciliation-strategies-experience.md)** - Auto-matched items, review suggested matches, manual match creation, reject false positives, split transaction matching, adjust thresholds
- **[4.1 Reminders Experience](ux-flows/4.1-reminders-experience.md)** - Create alert rules, receive notifications, snooze/dismiss, configure channels, view notification history, test conditions
- **[4.2 Forecast Experience](ux-flows/4.2-forecast-experience.md)** - View projections, compare algorithms, create goals, track progress, analyze burn rate, run scenarios
- **[4.3 Corrections Experience](ux-flows/4.3-corrections-experience.md)** - Correct single field, view audit trail, revert override, bulk corrections, handle validation errors, concurrent edit warnings
- **[5.1 Provenance Experience](ux-flows/5.1-provenance-experience.md)** - Run "as of" query, make retroactive correction, view bitemporal timeline, export audit report, handle validation errors, compare snapshots, schedule future-dated change
- **[5.2 Schema Registry Experience](ux-flows/5.2-schema-registry-experience.md)** - Publish schema version, detect breaking changes, execute migration with rollback, compare versions, approve breaking change request, handle migration failure
- **[5.3 Rule Performance & Logs Experience](ux-flows/5.3-rule-performance-logs-experience.md)** - Identify slow parser, optimize normalization rules, investigate queue backlog, detect performance degradation, handle error spikes, capacity planning
- **[5.4 Security & Access Experience](ux-flows/5.4-security-access-experience.md)** - Security admin reviews dashboard, owner unmasks PII (with audit), admin assigns roles (temporary), auditor reviews logs (compliance report), accountant views masked data, security event triggered

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
| | 3.8 Cluster Rules | ✅ Complete |
| | 3.9 Reconciliation Strategies | ✅ Complete |
| **4. Derivatives** | 4.1 Reminders | ✅ Complete |
| | 4.2 Forecast | ✅ Complete |
| | 4.3 Corrections Flow | ✅ Complete |
| **5. Governance** | 5.1 Provenance Ledger | ✅ Complete |
| | 5.2-5.5 | 📝 Pending |

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
