# Finance App Requirements (Simple)

> **Purpose:** Personal finance application requirements - simple, practical, single-user scale

---

## 📋 Overview

This directory contains the **simple, practical requirements** for a personal finance application. These requirements are intentionally stripped of enterprise complexity and focus on what a real user needs for managing 500 transactions per month.

**Target user:** Individual managing personal finances (checking, savings, credit cards)
**Scale:** 500 transactions/month, 6 accounts, 20-30 categories
**Tech stack:** SQLite, filesystem, simple web app

---

## 📄 Requirements Documents

### [1. Upload and Process](1-upload-and-process.md)
- Upload bank PDFs (Bank of America)
- Automatic transaction extraction
- Data normalization
- **Scale:** 10 PDFs/month, 500 tx/month
- **Tech:** PyPDF2, SQLite, filesystem storage

### [2. View and Filter Transactions](2-view-and-filter-transactions.md)
- Transaction list with pagination
- Filter by date, account, category, amount
- Search by description
- **Scale:** 50-100 tx per page, 6,000 total transactions
- **Tech:** Simple SQL queries, offset pagination

### [3. Categorize Spending](3-categorize-spending.md)
- Auto-suggest categories based on merchant
- Manual category assignment
- Create custom categories
- Simple rule engine (merchant → category)
- **Scale:** 20-30 categories, 50-100 rules
- **Tech:** YAML config, substring matching

### [4. Reports and Exports](4-reports-and-exports.md)
- Monthly summary (income, expenses, net)
- Top spending categories
- CSV export
- **Scale:** Monthly reports, 500 rows per export
- **Tech:** SQL aggregations, Python CSV module

---

## 🎯 Key Principles

### Simple > Complex
- SQLite over PostgreSQL (500 tx/month doesn't need Postgres)
- Filesystem over S3 (10 PDFs/month doesn't need cloud storage)
- Synchronous over async (user can wait 30 seconds for PDF processing)
- Offset pagination over cursor (10 pages max, no performance issues)

### Practical > Theoretical
- No machine learning for categorization (simple rules work fine)
- No real-time updates (refresh page is OK)
- No multi-user access (single user app)
- No horizontal scaling (fits on one laptop)

### User-Focused > Feature-Focused
- Solve actual problems (upload statements, see spending)
- Not theoretical problems (10M transactions, distributed tracing)

---

## ❌ What's Explicitly OUT of Scope

**Enterprise features:**
- Multi-tenancy, role-based access control
- Horizontal sharding, read replicas
- Message queues, event streaming (Kafka)
- Microservices architecture
- Redis caching, ElasticSearch
- TimescaleDB, PostgreSQL partitioning
- Kubernetes, container orchestration

**Advanced features:**
- Machine learning categorization
- Budget forecasting, burn rate analysis
- Automatic bank sync (Plaid integration)
- Mobile apps (iOS, Android)
- Multi-currency support
- Tax preparation features
- Bill payment, money transfers

**Over-engineering:**
- Sub-100ms query performance (2 seconds is fine)
- 99.99% uptime SLA (personal app, downtime is OK)
- Distributed tracing, Prometheus metrics
- Blue-green deployments, canary releases

---

## 🔗 Relationship to Universal Abstractions

These simple requirements are implemented using the **universal abstractions** documented in [`/docs/systems/`](../systems/):

- Uses [StorageEngine](../systems/truth-construction/primitives/StorageEngine.md) for file storage (even though we only need filesystem)
- Uses [ProvenanceLedger](../systems/audit/primitives/ProvenanceLedger.md) for audit trail (even though we only need basic logging)
- Uses [Parser](../systems/truth-construction/primitives/Parser.md) interface (even though we only need one parser)

**The validation question:**
> Can a simple app (500 tx/month) be built using complex abstractions (10M+ scale)?

**Answer:** Yes, if the abstractions are truly universal and don't force complexity.

See: [Validation document](../validation/simple-app-uses-complex-abstractions.md)

---

## 📊 Scale Comparison

| Metric | Simple App | Enterprise Scale (primitives support) |
|--------|------------|--------------------------------------|
| Transactions/month | 500 | 10M+ |
| Storage | Filesystem | S3, distributed storage |
| Database | SQLite | PostgreSQL with sharding |
| Processing | Synchronous | Async with queues |
| Users | 1 | 1,000+ |
| Uptime | Best effort | 99.99% SLA |
| Query latency | <2 seconds | <100ms p95 |

**Key insight:** The primitives in `/docs/systems/` CAN handle enterprise scale, but the simple app doesn't NEED to use those features.

---

## 🚀 Implementation Path

1. ✅ Define simple requirements (this directory)
2. ✅ Define universal abstractions ([/docs/systems/](../systems/))
3. ⏳ Validate abstractions can build simple app
4. ⏳ Implement simple app using abstractions
5. ⏳ Test with real data (Bank of America PDFs)
6. ⏳ Deploy for single user

---

**Last Updated:** 2025-10-27
**Status:** Requirements defined, ready for validation
