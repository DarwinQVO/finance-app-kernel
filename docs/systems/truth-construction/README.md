# Truth Construction System

**Status**: Universal / Domain-Agnostic
**Purpose**: Transform raw artifacts into verified canonical records through a four-layer pipeline

---

## Overview

The Truth Construction System is a universal data processing pipeline that transforms unstructured documents (PDFs, CSVs, images, etc.) into queryable, verified canonical records. It works identically across ALL domains: finance, healthcare, legal, research, manufacturing, and more.

## Architecture

The system consists of four sequential layers:

```
Raw Files → Storage → Extract → Normalize → Query → Canonical Records
```

### Layer 1: Storage Layer
**Purpose**: Content-addressable blob storage with deduplication

**Primitives:**
- [StorageEngine](primitives/StorageEngine.md) - SHA-256 hash-based content-addressable storage
- [FileArtifact](primitives/FileArtifact.md) - Metadata wrapper for stored files
- [HashCalculator](primitives/HashCalculator.md) - SHA-256 hash computation for deduplication

**Input**: Raw files (any format)
**Output**: Stored artifacts with immutable references (hash-based)

---

### Layer 2: Extraction Layer
**Purpose**: Parse raw artifacts into raw observations (AS-IS, no validation)

**Primitives:**
- [Parser](primitives/Parser.md) - Universal parser interface (pluggable)
- [ObservationStore](primitives/ObservationStore.md) - Raw data persistence
- [ParserRegistry](primitives/ParserRegistry.md) - Service discovery for parsers
- [ParseLog](primitives/ParseLog.md) - Execution log for observability
- [ParserVersionManager](primitives/ParserVersionManager.md) - Semantic versioning for parsers
- [ParserSelector](primitives/ParserSelector.md) - Smart parser selection logic
- [UploadRecord](primitives/UploadRecord.md) - State machine orchestrator for upload pipeline

**Input**: Stored artifacts (hash references)
**Output**: Raw observations (unvalidated, AS-IS)

**Philosophy**: Extract first, validate later. Preserve original data for re-normalization.

---

### Layer 3: Normalization Layer
**Purpose**: Validate and normalize raw observations into canonical records

**Primitives:**
- [Normalizer](primitives/Normalizer.md) - Universal normalizer interface (pluggable)
- [ValidationEngine](primitives/ValidationEngine.md) - Rule-based validation
- [CanonicalStore](primitives/CanonicalStore.md) - Normalized data persistence
- [NormalizationRuleSet](primitives/NormalizationRuleSet.md) - Externalized validation rules
- [NormalizationLog](primitives/NormalizationLog.md) - Execution log for observability

**Input**: Raw observations
**Output**: Canonical records (validated, normalized)

**Philosophy**: Validation is a separate stage from extraction, enabling iterative refinement.

---

### Layer 4: Query Layer
**Purpose**: Provide efficient access to canonical records

**Primitives:**
- [TransactionQuery](primitives/TransactionQuery.md) - Fluent query builder for canonical records
- [PaginationEngine](primitives/PaginationEngine.md) - Cursor-based pagination (efficient)
- [IndexStrategy](primitives/IndexStrategy.md) - Index optimization (compound indexes)
- [ExportEngine](primitives/ExportEngine.md) - Export to multiple formats (CSV, JSON, Excel)
- [ArtifactRetriever](primitives/ArtifactRetriever.md) - Retrieve original artifacts by reference

**Input**: Query parameters (filters, sorting, pagination)
**Output**: Paginated canonical records or exported files

---

## Multi-Domain Applicability

The Truth Construction System works identically across domains:

| Domain | Raw Input | Observations | Canonical Records |
|--------|-----------|--------------|-------------------|
| **Finance** | Bank statements (PDF, CSV) | Raw transactions | Canonical transactions |
| **Healthcare** | Lab reports (PDF, HL7) | Raw lab results | Canonical lab results |
| **Legal** | Contracts (PDF, DOCX) | Raw clauses | Canonical clauses |
| **Research (RSRCH)** | Web pages (HTML, JSON) | Raw facts | Canonical facts |
| **E-commerce** | Supplier catalogs (CSV, XML) | Raw products | Canonical products |
| **Manufacturing** | Sensor data (CSV, JSON) | Raw readings | Canonical readings |

---

## Key Principles

1. **Domain-Agnostic**: No finance-specific code in this system
2. **Pluggable Parsers**: Add new parsers without modifying core logic
3. **Extract-Then-Validate**: Preserve original data for re-normalization
4. **Content-Addressable**: Immutable references via SHA-256 hashes
5. **Event-Sourced**: All operations logged to ProvenanceLedger (Audit System)

---

## Dependencies

- **Audit System** (cross-cutting): ProvenanceLedger logs all operations
- **API/Auth System** (cross-cutting): APIGateway provides secure access

---

## Related Documentation

- [Audit System](../audit/README.md) - Bitemporal audit trail
- [API/Auth System](../api-auth/README.md) - Secure API access
- [Interface Library](../../libraries/interface/README.md) - Reusable UI components
- [Finance App Example](../../verticals/README.md) - Complete application using this system

---

**Primitives Count**: 20
**Status**: Production-ready (all primitives include Domain Validation)
**Last Updated**: 2025-10-27
