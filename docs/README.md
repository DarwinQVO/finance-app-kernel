# Truth Construction Primitives - Architecture Documentation

> **Multi-domain truth construction systems demonstrated through finance domain instantiation.**

---

## 🏗️ Architecture Overview

This repository documents **3 independent systems + 1 reusable library** that work together to construct verifiable truth from raw artifacts. The finance application is used as a concrete example to demonstrate universal patterns.

```
┌─────────────────────────────────────────────────┐
│           FINANCE APP (Example)                 │
│  Uses: Truth Construction + Audit + API + IL   │
│  Adds: Domain-specific Registries (3.1-3.9)    │
└─────────────────────────────────────────────────┘
                    ▲
        ┌───────────┼───────────┐
        │           │           │
┌───────▼──────┐ ┌──▼───────┐ ┌▼──────────┐
│ Truth        │ │ Audit    │ │ API/Auth  │
│ Construction │ │ System   │ │ System    │
│ System       │ │          │ │           │
│              │ │          │ │           │
│ • Storage    │ │ • Prov   │ │ • Gateway │
│ • Extract    │ │ • Bitmp  │ │ • OAuth2  │
│ • Normalize  │ │ • Timeline│ │ • Access │
│ • Query      │ │          │ │           │
└──────────────┘ └──────────┘ └───────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│         Interface Library (IL)                  │
│  57 reusable UI components                      │
└─────────────────────────────────────────────────┘
```

### System 1: Truth Construction System
**Purpose**: Transform raw artifacts into verified canonical records
**Location**: [systems/truth-construction/](systems/truth-construction/)
**Domain-Agnostic**: ✅ Yes
**Primitives**: 20

Upload → Extract → Normalize → Query pipeline that works identically across finance, healthcare, legal, research, and any document-processing domain.

### System 2: Audit System (Provenance)
**Purpose**: Provide bitemporal audit trail for all operations
**Location**: [systems/audit/](systems/audit/)
**Domain-Agnostic**: ✅ Yes
**Primitives**: 5

Independent event store tracking "what we knew when" and "what was true when" across all systems.

### System 3: API/Auth System
**Purpose**: Secure API access with authentication and authorization
**Location**: [systems/api-auth/](systems/api-auth/)
**Domain-Agnostic**: ✅ Yes
**Primitives**: 8

Gateway, OAuth2, RBAC, metrics, and observability for any application.

### Library: Interface Library (IL)
**Purpose**: Reusable UI components for any domain
**Location**: [libraries/interface/](libraries/interface/)
**Domain-Agnostic**: ✅ Yes (patterns universal, vocabulary configurable)
**Components**: 57

React/TypeScript components for upload, query, audit, security, and domain management.

### Application Example: Finance App
**Purpose**: Concrete instantiation demonstrating universal patterns
**Location**: [verticals/](verticals/) + [primitives/ol/](primitives/ol/) (domain-specific)
**Domain-Specific**: ✅ Finance (uses all 3 systems + library + adds domain registries)
**Verticals**: 23

Complete personal finance application showing how universal systems compose into domain-specific applications.

---

## 📖 Quick Navigation

| If you want to... | Start here |
|-------------------|------------|
| **Understand the core data processing pipeline** | [Truth Construction System](systems/truth-construction/README.md) |
| **Learn about audit trail and temporal queries** | [Audit System](systems/audit/README.md) |
| **Set up authentication and API security** | [API/Auth System](systems/api-auth/README.md) |
| **Build UIs with reusable components** | [Interface Library](libraries/interface/README.md) |
| **See a complete application example** | [Finance App Verticals](#finance-app-example-23-verticals) |

---

## 🎯 Truth Construction System

**Purpose**: Universal data processing pipeline (Upload → Extract → Normalize → Query)

**20 Primitives organized in 4 layers:**

### Storage Layer
- [StorageEngine](systems/truth-construction/primitives/StorageEngine.md) - Content-addressable storage with deduplication
- [FileArtifact](systems/truth-construction/primitives/FileArtifact.md) - Metadata wrapper for stored files
- [HashCalculator](systems/truth-construction/primitives/HashCalculator.md) - SHA-256 hash computation

### Extraction Layer
- [Parser](systems/truth-construction/primitives/Parser.md) - Universal parser interface (pluggable)
- [ObservationStore](systems/truth-construction/primitives/ObservationStore.md) - Raw data persistence (AS-IS)
- [ParserRegistry](systems/truth-construction/primitives/ParserRegistry.md) - Service discovery
- [ParseLog](systems/truth-construction/primitives/ParseLog.md) - Execution log
- [ParserVersionManager](systems/truth-construction/primitives/ParserVersionManager.md) - Version management
- [ParserSelector](systems/truth-construction/primitives/ParserSelector.md) - Auto-selection logic
- [UploadRecord](systems/truth-construction/primitives/UploadRecord.md) - State machine orchestrator

### Normalization Layer
- [Normalizer](systems/truth-construction/primitives/Normalizer.md) - Universal normalizer interface
- [ValidationEngine](systems/truth-construction/primitives/ValidationEngine.md) - Rule validation
- [CanonicalStore](systems/truth-construction/primitives/CanonicalStore.md) - Normalized data persistence
- [NormalizationRuleSet](systems/truth-construction/primitives/NormalizationRuleSet.md) - Externalized rules
- [NormalizationLog](systems/truth-construction/primitives/NormalizationLog.md) - Execution log

### Query Layer
- [TransactionQuery](systems/truth-construction/primitives/TransactionQuery.md) - Fluent query builder
- [PaginationEngine](systems/truth-construction/primitives/PaginationEngine.md) - Cursor-based pagination
- [IndexStrategy](systems/truth-construction/primitives/IndexStrategy.md) - Index optimization
- [ExportEngine](systems/truth-construction/primitives/ExportEngine.md) - Multi-format export
- [ArtifactRetriever](systems/truth-construction/primitives/ArtifactRetriever.md) - Artifact retrieval

📚 **[Full Documentation →](systems/truth-construction/README.md)**

---

## 🔍 Audit System (Provenance)

**Purpose**: Bitemporal audit trail and temporal queries

**5 Primitives:**

- [ProvenanceLedger](systems/audit/primitives/ProvenanceLedger.md) - Append-only bitemporal event store
- [BitemporalQuery](systems/audit/primitives/BitemporalQuery.md) - Query by transaction_time or valid_time
- [TimelineReconstructor](systems/audit/primitives/TimelineReconstructor.md) - Timeline visualization
- [RetroactiveCorrector](systems/audit/primitives/RetroactiveCorrector.md) - Retroactive corrections
- [AuditLog](systems/audit/primitives/AuditLog.md) - Structured audit logging

📚 **[Full Documentation →](systems/audit/README.md)**

---

## 🔐 API/Auth System

**Purpose**: Secure API access with authentication and authorization

**8 Primitives:**

- [APIGateway](systems/api-auth/primitives/APIGateway.md) - Request routing, rate limiting
- [APIRouter](systems/api-auth/primitives/APIRouter.md) - Route matching and dispatch
- [OAuth2Provider](systems/api-auth/primitives/OAuth2Provider.md) - OAuth2 flows
- [APIKeyValidator](systems/api-auth/primitives/APIKeyValidator.md) - API key management
- [AccessControl](systems/api-auth/primitives/AccessControl.md) - RBAC
- [MetricsCollector](systems/api-auth/primitives/MetricsCollector.md) - Prometheus metrics
- [DashboardEngine](systems/api-auth/primitives/DashboardEngine.md) - Dashboards
- [SchemaRegistry](systems/api-auth/primitives/SchemaRegistry.md) - JSON schema validation

📚 **[Full Documentation →](systems/api-auth/README.md)**

---

## 🎨 Interface Library (IL)

**Purpose**: Reusable UI components for any domain

**57 Components organized by category:**

- **Upload & Ingestion** (7): FileUpload, FacturaUploadDialog, QueueMonitorPanel, ParserSelectorDialog, etc.
- **Data Display & Query** (8): TransactionTable, DashboardGrid, MetricCard, SavedViewSelector, etc.
- **Audit & Provenance** (5): AuditTrailViewer, ProvenanceTimeline, RetroactiveCorrectionDialog, etc.
- **Domain Management** (15): AccountManager, CounterpartyManager, SeriesManager, TaxCategoryManager, etc.
- **Normalization & Matching** (6): RuleEditorDialog, ClusterViewer, MatchReviewDialog, etc.
- **Forecasting & Goals** (6): ForecastChart, GoalProgressCard, RecurrenceConfigDialog, etc.
- **Reconciliation** (2): ReconciliationDashboard, NotificationPanel
- **Security & Access** (4): SecurityDashboard, RoleManager, APIKeyManagementPanel, etc.
- **Schema & Observability** (4): SchemaEditor, PerformanceDashboard, CompatibilityViewer, etc.

📚 **[Full Documentation →](libraries/interface/README.md)**

---

## 💼 Finance App Example (23 Verticals)

**Purpose**: Complete application demonstrating how universal systems compose into domain-specific solutions

> **📖 [CANONICAL VERTICAL LIST](../VERTICALS.md)** — Single source of truth for all 23 verticals

### Group 1: Upload & Ingestion (Complete ✅)
- **[1.1 Upload Flow](verticals/1.1-upload-flow.md)** - File upload with deduplication and state machine
- **[1.2 Extraction](verticals/1.2-extraction.md)** - Parser execution, raw observation extraction (AS-IS)
- **[1.3 Normalization](verticals/1.3-normalization.md)** - Raw → canonical transformation, validation, categorization

### Group 2: Exploration & Visualization
- **[2.1 Transaction List View](verticals/2.1-transaction-list-view.md)** ✅ - Pagination, filtering, sorting, export
- **[2.2 OL Exploration](verticals/2.2-ol-exploration.md)** ✅ - Drill-down, decisions, provenance, artifact viewing
- **[2.3 Finance Dashboard](verticals/2.3-finance-dashboard.md)** ✅ - Saved views, aggregate metrics, PDF exports

### Group 3: Registries (Finance Domain-Specific)
- **[3.1 Account Registry](verticals/3.1-account-registry.md)** ✅ - Closed registry, CRUD, soft delete
- **[3.2 Counterparty Registry](verticals/3.2-counterparty-registry.md)** ✅ - Open registry, fuzzy matching, merge
- **[3.3 Series Registry](verticals/3.3-series-registry.md)** ✅ - Recurring payments, variance detection
- **[3.4 Tax Categorization](verticals/3.4-tax-categorization.md)** ✅ - Multi-jurisdiction, auto-classification
- **[3.5 Relationships](verticals/3.5-relationships.md)** ✅ - Transfer detection, FX conversion
- **[3.6 Unit](verticals/3.6-unit.md)** ✅ - Multi-currency, date/timezone normalization
- **[3.7 Parser Registry](verticals/3.7-parser-registry.md)** ✅ - Parser discovery, versioning
- **[3.8 Cluster Rules](verticals/3.8-cluster-rules.md)** ✅ - Merchant normalization, fuzzy matching
- **[3.9 Reconciliation Strategies](verticals/3.9-reconciliation-strategies.md)** ✅ - Multi-source matching

### Group 4: Derivatives & Insights
- **[4.1 Reminders](verticals/4.1-reminders.md)** ✅ - Alert rules, notification delivery
- **[4.2 Forecast](verticals/4.2-forecast.md)** ✅ - Projections, goal tracking, burn rate
- **[4.3 Corrections Flow](verticals/4.3-corrections-flow.md)** ✅ - Field overrides, audit trail

### Group 5: Governance & Meta
- **[5.1 Provenance Ledger](verticals/5.1-provenance-ledger.md)** ✅ - Bitemporal tracking, retroactive corrections

**Finance-Specific Primitives** (in [primitives/ol/](primitives/ol/)):
- AccountStore, CounterpartyStore, SeriesStore, TaxCategoryStore, RelationshipStore
- CurrencyConverter, DateNormalizer, StringNormalizer, MerchantNormalizer
- ReconciliationEngine, ClusteringEngine, FuzzyMatcher
- And 45+ more domain-specific primitives

---

## 📐 JSON Schemas

Executable contracts extracted from vertical specifications:

**Location**: [schemas/](schemas/)

**78 schemas** covering:
- Upload records, observations, canonical transactions
- Query responses, drill-down data, decision explanations
- Account, counterparty, series, tax category entities
- Relationship candidates, FX details, reconciliation results
- Alert rules, forecasts, goal configs, field overrides
- Provenance records, bitemporal queries, timeline events
- Schema versions, migration plans, compatibility reports
- Execution metrics, queue snapshots, audit events

📚 **[Full Schema List →](schemas/)**

---

## 🏛️ Architecture Decision Records (ADR)

Key architectural decisions with rationale:

**Location**: [adr/](adr/)

**38 ADRs** covering:
- Upload & Ingestion (ADR-0001 to ADR-0005)
- Exploration & Visualization (ADR-0006 to ADR-0008)
- Registries (ADR-0009 to ADR-0017)
- Derivatives & Insights (ADR-0018 to ADR-0029)
- Governance & Meta (ADR-0030 to ADR-0038)

📚 **[Full ADR List →](adr/)**

---

## 🎨 UX Flows

User experience specifications with wireframes and journeys:

**Location**: [ux-flows/](ux-flows/)

**23 UX flows** covering all verticals:
- Upload experience, extraction feedback, normalization results
- Transaction list filtering, drill-down interactions
- Registry management (accounts, counterparties, series, tax categories)
- Relationship detection, currency management, parser selection
- Alert configuration, forecast visualization, corrections workflow
- Provenance queries, schema migrations, security dashboards

📚 **[Full UX Flow List →](ux-flows/)**

---

## 📊 Progress Summary

| Component | Count | Status |
|-----------|-------|--------|
| **Verticals** | 23 | ✅ Complete |
| **Truth Construction Primitives** | 20 | ✅ Complete |
| **Audit System Primitives** | 5 | ✅ Complete |
| **API/Auth System Primitives** | 8 | ✅ Complete |
| **Finance Domain Primitives** | 58 | ✅ Complete |
| **Interface Library Components** | 57 | ✅ Complete |
| **JSON Schemas** | 78 | ✅ Complete |
| **ADRs** | 38 | ✅ Complete |
| **UX Flows** | 23 | ✅ Complete |

**Total Documentation Artifacts**: 310

---

## 🧠 Key Architectural Insights

### Q1: Is "Objective Layer" a Layer or a System?
**Answer**: **System** (renamed to "Truth Construction System")

The "Objective Layer" is not a horizontal layer - it's a vertical system containing multiple layers (Storage, Extract, Normalize, Query).

### Q2: Is Storage independent or part of Truth Construction?
**Answer**: **Layer within Truth Construction** (bottom layer)

While StorageEngine has no dependencies, it's architecturally the foundation layer of the Truth Construction pipeline.

### Q3: Where does ProvenanceLedger live?
**Answer**: **Independent Audit System** (cross-cutting)

ProvenanceLedger is a separate system that provides audit services TO Truth Construction (and other systems). It's not "part of" Truth Construction - it audits it.

### Q4: What are the actual System boundaries?
**Answer**: **3 Systems + 1 Library + Application Examples**

- **System 1**: Truth Construction (Storage → Extract → Normalize → Query)
- **System 2**: Audit (ProvenanceLedger, BitemporalQuery, Timeline, Corrections)
- **System 3**: API/Auth (Gateway, OAuth2, RBAC, Metrics)
- **Library**: Interface Library (57 reusable UI components)
- **Application**: Finance App (domain-specific instantiation)

📚 **[Full Architectural Analysis →](/tmp/architectural_analysis.md)**

---

## 📦 Repository Philosophy

This repository contains **only specification artifacts** - the crystallized output of vertical analysis using literate programming principles.

**What's included:**
- ✅ Vertical specifications (`.md`)
- ✅ System documentation (Truth Construction, Audit, API/Auth)
- ✅ Primitive specifications (OL/IL `.md`)
- ✅ JSON schemas (`.schema.json`)
- ✅ ADRs (Architecture Decision Records)
- ✅ UX flows and wireframes

**What's excluded:**
- ❌ Implementation code (lives elsewhere)
- ❌ Internal coordination files
- ❌ Status reports

---

## 🚀 Navigation by Role

- **For Product/Design:** Start with [verticals/](verticals/) and [ux-flows/](ux-flows/)
- **For Engineering:** Read [systems/truth-construction/](systems/truth-construction/) → [primitives/](systems/) → [schemas/](schemas/)
- **For Architecture:** Focus on [adr/](adr/) and system READMEs
- **For UI Development:** Explore [libraries/interface/](libraries/interface/)

---

*Last Updated: 2025-10-27*
*Architecture Version: 1.0 (Separated Systems)*
