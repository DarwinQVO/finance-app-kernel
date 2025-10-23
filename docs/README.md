# Truth Construction Primitives - Finance Instantiation

> **Literate Programming approach:** Multi-domain truth construction primitives demonstrated through finance domain. Each vertical specification mixes prose (WHY), architecture (HOW), and executable contracts (WHAT) in a single source of truth.

---

## 📋 Vertical Specifications

### Group 1: Upload & Ingestion
- **[1.1 Upload Flow](verticals/1.1-upload-flow.md)** - Complete upload specification with state machine, contracts, and primitives

*Pending: 1.2 Extraction, 1.3 Normalization*

---

## 🧩 Primitives Catalog

### Objective Layer (OL)
**Multi-domain truth construction primitives** (finance instantiation)

These primitives are domain-agnostic - they construct verifiable truth across ANY domain (finance, medicine, legal, etc.). This repository demonstrates their instantiation in the finance domain.

- **[StorageEngine](primitives/ol/StorageEngine.md)** - Content-addressable storage with deduplication
- **[ProvenanceLedger](primitives/ol/ProvenanceLedger.md)** - Append-only audit trail with cryptographic integrity
- **[FileArtifact](primitives/ol/FileArtifact.md)** - Internal metadata wrapper for uploaded files
- **[HashCalculator](primitives/ol/HashCalculator.md)** - Streaming SHA-256 hash calculator
- **[UploadRecord](primitives/ol/UploadRecord.md)** - State machine orchestrator for upload flow

### Interface Layer (IL)
Reusable UI components:

- **[FileUpload](primitives/il/FileUpload.md)** - Drag & drop component with validation
- **[IL Components Summary](primitives/il/_IL_COMPONENTS_SUMMARY.md)** - Catalog of all IL components

---

## 📐 JSON Schemas

Executable contracts extracted from vertical specifications:

- **[upload-record.schema.json](schemas/upload-record.schema.json)** - UploadRecord state machine contract
- **[provenance-entry.schema.json](schemas/provenance-entry.schema.json)** - ProvenanceLedger entry format
- **[upload-error-response.schema.json](schemas/upload-error-response.schema.json)** - Standardized error responses

---

## 🏛️ Architecture Decision Records (ADR)

Key architectural decisions with rationale:

- **[ADR-0001: Canonical ID Decision](adr/0001-canonical-id-decision.md)** - Why `upload_id` not `file_id`
- **[ADR-0002: State Machine Global](adr/0002-state-machine-global.md)** - Why single `status` field
- **[ADR-0003: Runner/Coordinator Split](adr/0003-runner-coordinator-split.md)** - Separation of concerns

---

## 🎨 UX Flows

User experience specifications with wireframes and journeys:

- **[1.1 Upload Experience](ux-flows/1.1-upload-experience.md)** - Complete user journeys, wireframes, and decision points

---

## 📊 Progress

| Group | Vertical | Status |
|-------|----------|--------|
| **1. Upload & Ingestion** | 1.1 Upload Flow | ✅ Complete |
| | 1.2 Extraction | 📝 Pending |
| | 1.3 Normalization | 📝 Pending |
| **2. Exploration & Viz** | 2.1-2.3 | 📝 Pending |
| **3. Registries** | 3.1-3.9 | 📝 Pending |
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
