# Interface Library (IL)

**Status**: Universal / Domain-Agnostic
**Purpose**: Reusable UI components for building applications on top of any domain system

---

## Overview

The Interface Library is a collection of 57 React/TypeScript UI components that can be incorporated into any application. While examples use finance vocabulary (transactions, accounts), the patterns are universal and work across all domains.

## Component Categories

### Upload & Ingestion (7 components)
Upload raw artifacts into the Truth Construction System:
- [FileUpload](components/FileUpload.md) - Drag-and-drop file uploader
- [FacturaUploadDialog](components/FacturaUploadDialog.md) - Multi-file upload with metadata
- [QueueMonitorPanel](components/QueueMonitorPanel.md) - Monitor upload/parse/normalize queue
- [ParserSelectorDialog](components/ParserSelectorDialog.md) - Select parser for upload
- [ParserCapabilitiesCard](components/ParserCapabilitiesCard.md) - Show parser capabilities
- [ParserVersionDropdown](components/ParserVersionDropdown.md) - Select parser version
- [MigrationWizard](components/MigrationWizard.md) - Guided data migration flow

### Data Display & Query (8 components)
Display and query canonical records from the Truth Construction System:
- [TransactionTable](components/TransactionTable.md) - Sortable, filterable table
- [DashboardGrid](components/DashboardGrid.md) - Responsive card grid layout
- [MetricCard](components/MetricCard.md) - KPI card with trend indicator
- [SavedViewSelector](components/SavedViewSelector.md) - Load saved queries
- [DrillDownPanel](components/DrillDownPanel.md) - Hierarchical data exploration
- [AsOfQueryBuilder](components/AsOfQueryBuilder.md) - Build temporal queries
- [TimelineViewer](components/TimelineViewer.md) - Visualize event timelines
- [AmountDisplayCard](components/AmountDisplayCard.md) - Format currency/numbers

### Audit & Provenance (5 components)
Visualize audit trail from the Audit System:
- [AuditTrailViewer](components/AuditTrailViewer.md) - Show complete audit trail
- [ProvenanceTimeline](components/ProvenanceTimeline.md) - Bitemporal timeline
- [AuditViewer](components/AuditViewer.md) - Search and filter audit logs
- [RetroactiveCorrectionDialog](components/RetroactiveCorrectionDialog.md) - Perform corrections
- [CorrectionDialog](components/CorrectionDialog.md) - Simple correction form

### Domain Management (15 components)
Manage domain-specific registries (accounts, counterparties, etc.):
- [AccountManager](components/AccountManager.md) - CRUD for accounts
- [AccountSelector](components/AccountSelector.md) - Dropdown account selector
- [CounterpartyManager](components/CounterpartyManager.md) - CRUD for counterparties
- [CounterpartySelector](components/CounterpartySelector.md) - Dropdown counterparty selector
- [SeriesManager](components/SeriesManager.md) - CRUD for recurring series
- [SeriesSelector](components/SeriesSelector.md) - Dropdown series selector
- [TaxCategoryManager](components/TaxCategoryManager.md) - CRUD for tax categories
- [TaxCategorySelector](components/TaxCategorySelector.md) - Dropdown tax category selector
- [RelationshipPanel](components/RelationshipPanel.md) - Manage entity relationships
- [MergeCounterpartiesDialog](components/MergeCounterpartiesDialog.md) - Merge duplicate entities
- [TransferLinkDialog](components/TransferLinkDialog.md) - Link transfers between accounts
- [CurrencySelectorDialog](components/CurrencySelectorDialog.md) - Multi-currency picker
- [ExchangeRateWidget](components/ExchangeRateWidget.md) - Display exchange rates
- [FXConversionCard](components/FXConversionCard.md) - Convert between currencies
- [FieldOverrideIndicator](components/FieldOverrideIndicator.md) - Show field overrides

### Normalization & Matching (6 components)
Configure normalization rules and review matches:
- [RuleEditorDialog](components/RuleEditorDialog.md) - Create/edit normalization rules
- [RuleOptimizer](components/RuleOptimizer.md) - Optimize rule performance
- [ClusterViewer](components/ClusterViewer.md) - View merchant clusters
- [MatchReviewDialog](components/MatchReviewDialog.md) - Review fuzzy matches
- [ManualMatchDialog](components/ManualMatchDialog.md) - Manually match entities
- [MerchantRulesManager](components/MerchantRulesManager.md) - Manage merchant rules

### Forecasting & Goals (5 components)
Plan future transactions and track goals:
- [ForecastChart](components/ForecastChart.md) - Visualize projected transactions
- [GoalProgressCard](components/GoalProgressCard.md) - Track goal completion
- [GoalConfigDialog](components/GoalConfigDialog.md) - Configure goals
- [RecurrenceConfigDialog](components/RecurrenceConfigDialog.md) - Define recurrence patterns
- [ReminderConfigDialog](components/ReminderConfigDialog.md) - Set up reminders
- [ReminderBell](components/ReminderBell.md) - Notification bell icon

### Reconciliation (2 components)
Match transactions across systems:
- [ReconciliationDashboard](components/ReconciliationDashboard.md) - Main reconciliation UI
- [NotificationPanel](components/NotificationPanel.md) - Show notifications

### Security & Access Control (4 components)
Manage users, roles, and API access:
- [SecurityDashboard](components/SecurityDashboard.md) - Security settings overview
- [RoleManager](components/RoleManager.md) - CRUD for roles/permissions
- [APIKeyManagementPanel](components/APIKeyManagementPanel.md) - Manage API keys
- [WebhookManagementPanel](components/WebhookManagementPanel.md) - Configure webhooks

### Schema & Observability (5 components)
Manage schemas and monitor performance:
- [SchemaEditor](components/SchemaEditor.md) - Edit JSON schemas
- [PerformanceDashboard](components/PerformanceDashboard.md) - Monitor system performance
- [CompatibilityViewer](components/CompatibilityViewer.md) - Check schema compatibility

---

## Multi-Domain Applicability

While examples use finance vocabulary, components adapt to any domain:

| Component | Finance Example | Healthcare Example | Legal Example | RSRCH Example |
|-----------|-----------------|---------------------|---------------|---------------|
| **TransactionTable** | Bank transactions | Lab results | Invoices | Research facts |
| **CounterpartyManager** | Merchants/banks | Healthcare providers | Clients/vendors | Web sources |
| **AccountManager** | Bank accounts | Patient accounts | Matter codes | Research projects |
| **RuleEditorDialog** | Normalization rules | Lab result rules | Contract rules | Fact validation rules |
| **AuditTrailViewer** | Transaction history | Lab amendment history | Contract lifecycle | Fact correction history |

**Pattern**: Component logic is universal, vocabulary is configurable.

---

## Technology Stack

- **Framework**: React 18+ with TypeScript
- **Styling**: Tailwind CSS (utility-first)
- **State Management**: React Context + Hooks
- **Forms**: React Hook Form + Zod validation
- **Charts**: Recharts (lightweight, composable)
- **Icons**: Heroicons (MIT licensed)

---

## Key Features

### 1. Domain-Agnostic Patterns
- Components accept generic data structures
- Labels and vocabulary configurable via props
- No hardcoded finance-specific logic

### 2. Composition-First
- Small, focused components that compose together
- Example: `TransactionTable` + `DrillDownPanel` = Hierarchical explorer

### 3. Accessible (a11y)
- Keyboard navigation support
- ARIA labels for screen readers
- Focus management

### 4. Responsive Design
- Mobile-first approach
- Breakpoint-aware layouts
- Touch-friendly controls

### 5. Performance Optimized
- Virtual scrolling for large tables (>1000 rows)
- Lazy loading for images/charts
- Memoization for expensive calculations

---

## Integration with Systems

### Truth Construction System
- **Upload components** → StorageEngine, Parser
- **Query components** → TransactionQuery, PaginationEngine
- **Export components** → ExportEngine

### Audit System
- **Audit components** → ProvenanceLedger, BitemporalQuery
- **Timeline components** → TimelineReconstructor
- **Correction components** → RetroactiveCorrector

### API/Auth System
- **Security components** → OAuth2Provider, AccessControl
- **API management components** → APIGateway, APIKeyValidator

---

## Usage Example

```tsx
import { FileUpload, TransactionTable, AuditTrailViewer } from '@truth-construction/interface-library';

function MyApp() {
  return (
    <div>
      {/* Upload raw files */}
      <FileUpload onUploadComplete={handleUpload} />

      {/* Display canonical records */}
      <TransactionTable
        data={transactions}
        onRowClick={handleDrillDown}
      />

      {/* Show audit trail */}
      <AuditTrailViewer entityId="TX_123" />
    </div>
  );
}
```

---

## Key Principles

1. **Reusable**: Works across any domain application
2. **Composable**: Small components that combine into complex UIs
3. **Accessible**: WCAG 2.1 AA compliant
4. **Type-Safe**: Full TypeScript support
5. **Documented**: Every component has examples and API docs

---

## Related Documentation

- [Truth Construction System](../../systems/truth-construction/README.md) - Backend data processing
- [Audit System](../../systems/audit/README.md) - Audit trail backend
- [API/Auth System](../../systems/api-auth/README.md) - API security
- [Finance App Example](../../verticals/README.md) - Complete application using IL components

---

**Components Count**: 57
**Status**: Production-ready (all components have multi-domain examples)
**Last Updated**: 2025-10-27
