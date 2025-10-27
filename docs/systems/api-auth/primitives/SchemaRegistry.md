# SchemaRegistry OL Primitive

**Domain:** Schema Registry (Vertical 5.2)
**Layer:** Objective Layer (OL)
**Version:** 1.0.0
**Status:** Specification

---

## Table of Contents

1. [Overview](#overview)
2. [Purpose & Scope](#purpose--scope)
3. [Multi-Domain Applicability](#multi-domain-applicability)
4. [Core Concepts](#core-concepts)
5. [Interface Definition](#interface-definition)
6. [Data Model](#data-model)
7. [Core Functionality](#core-functionality)
8. [Versioning Strategy](#versioning-strategy)
9. [Schema Evolution](#schema-evolution)
10. [Storage & Indexing](#storage--indexing)
11. [Edge Cases](#edge-cases)
12. [Performance Characteristics](#performance-characteristics)
13. [Implementation Notes](#implementation-notes)
14. [Security Considerations](#security-considerations)
15. [Integration Patterns](#integration-patterns)
16. [Multi-Domain Examples](#multi-domain-examples)
17. [Testing Strategy](#testing-strategy)
18. [Migration Guide](#migration-guide)
19. [Related Primitives](#related-primitives)
20. [References](#references)

---

## Overview

The **SchemaRegistry** is a centralized, version-controlled repository for managing schema definitions across distributed systems. It provides semantic versioning, backward compatibility checking, and efficient schema retrieval to ensure data contract consistency across services, APIs, and data pipelines.

### Key Capabilities

- **Centralized Schema Storage**: Single source of truth for all schema definitions
- **Semantic Versioning**: Major.minor.patch versioning with upgrade/downgrade support
- **UNIQUE Constraint**: Enforces (schema_id, version) uniqueness at database level
- **Fast Retrieval**: O(1) lookups for latest version, O(log n) for version history
- **JSON Schema Support**: Stores schemas as JSONB with full validation
- **Search & Discovery**: Full-text search across schema names, descriptions, and fields
- **Evolution Tracking**: Complete history of schema changes with rationale

### Design Philosophy

The SchemaRegistry follows four core principles:

1. **Single Source of Truth**: All schema definitions centrally managed and versioned
2. **Backward Compatibility**: Breaking changes require major version bumps
3. **Discoverability**: Schemas are searchable by name, domain, and metadata
4. **Auditability**: Every schema version is immutable and traceable

---

## Purpose & Scope

### Problem Statement

In modern distributed systems, schema management poses critical challenges:

- **Data Contract Fragmentation**: Different services use incompatible schemas
- **Breaking Changes**: Schema updates break downstream consumers without warning
- **No Versioning**: Schema changes overwrite previous versions, losing history
- **Discovery Problem**: Developers can't find existing schemas, leading to duplication
- **Compliance Risk**: No audit trail of schema changes for regulatory requirements

Traditional approaches have significant limitations:

**Approach 1: Schema-in-Code**
- ❌ Schemas scattered across codebases
- ❌ No central version control
- ❌ Breaking changes undetected until runtime

**Approach 2: Schema-in-Database**
- ❌ Database-specific (not portable)
- ❌ No semantic versioning
- ❌ Limited validation capabilities

**Approach 3: Centralized Schema Registry**
- ✅ Single source of truth
- ✅ Semantic versioning with compatibility checks
- ✅ Full audit trail
- ✅ Language-agnostic (JSON Schema)
- ✅ Searchable and discoverable

### Solution

The SchemaRegistry implements a **centralized, version-controlled schema repository** with:

1. **PostgreSQL Storage**: JSONB column for schema definitions with UNIQUE constraint on (schema_id, version)
2. **Semantic Versioning**: Major.minor.patch versioning following semver conventions
3. **Immutability**: Published schemas cannot be modified, only deprecated
4. **Fast Retrieval**: Indexed queries for latest version and version history
5. **Full-Text Search**: Search schemas by name, description, tags, and field names
6. **Compatibility Enforcement**: Integration with BackwardCompatibilityChecker

---

## Multi-Domain Applicability

The SchemaRegistry is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Manage transaction schemas across banking systems, payment processors, and reporting pipelines.

**Schemas Managed**:
- Bank transaction format (v1.0.0 → v2.0.0 adds merchant_category_code)
- Payment request/response schemas
- Account statement schemas
- Reconciliation report schemas
- Tax document schemas (1099, W-2)

**Example**:
```typescript
// Register initial transaction schema
await registry.publish({
  schema_id: "bank_transaction",
  version: "1.0.0",
  schema: {
    type: "object",
    properties: {
      transaction_id: { type: "string" },
      date: { type: "string", format: "date-time" },
      amount: { type: "number" },
      merchant: { type: "string" },
      category: { type: "string" }
    },
    required: ["transaction_id", "date", "amount"]
  },
  description: "Bank transaction format for Chase statements",
  tags: ["finance", "banking", "transaction"]
});

// Evolve schema: add optional field (minor version)
await registry.publish({
  schema_id: "bank_transaction",
  version: "1.1.0",
  schema: {
    type: "object",
    properties: {
      transaction_id: { type: "string" },
      date: { type: "string", format: "date-time" },
      amount: { type: "number" },
      merchant: { type: "string" },
      category: { type: "string" },
      merchant_category_code: { type: "string" } // New optional field
    },
    required: ["transaction_id", "date", "amount"]
  },
  description: "Bank transaction format - added MCC field",
  tags: ["finance", "banking", "transaction"]
});
```

**Compliance**: SOX requires schema versioning for financial data integrity.

---

### 2. Healthcare

**Use Case**: Manage patient record schemas compliant with HL7 FHIR standards.

**Schemas Managed**:
- Patient demographics (FHIR R4)
- Lab result schemas (LOINC codes)
- Medication order schemas (RxNorm)
- Diagnostic report schemas
- Insurance claim schemas (X12 837)

**Example**:
```typescript
// Register FHIR Patient schema
await registry.publish({
  schema_id: "fhir_patient",
  version: "4.0.0",
  schema: {
    resourceType: "Patient",
    type: "object",
    properties: {
      id: { type: "string" },
      name: {
        type: "array",
        items: {
          type: "object",
          properties: {
            family: { type: "string" },
            given: { type: "array", items: { type: "string" } }
          }
        }
      },
      birthDate: { type: "string", format: "date" },
      gender: { type: "string", enum: ["male", "female", "other", "unknown"] }
    },
    required: ["id", "name"]
  },
  description: "FHIR R4 Patient resource schema",
  tags: ["healthcare", "fhir", "patient"]
});

// Search for all FHIR schemas
const fhirSchemas = await registry.searchSchemas({
  tags: ["fhir"]
});
```

**Compliance**: HIPAA requires data structure validation for PHI protection.

---

### 3. Legal

**Use Case**: Manage legal document schemas for e-discovery and case management.

**Schemas Managed**:
- Case filing schemas (court-specific)
- Evidence metadata schemas
- Deposition transcript schemas
- Contract template schemas
- Legal hold notification schemas

**Example**:
```typescript
// Register case filing schema
await registry.publish({
  schema_id: "court_filing",
  version: "1.0.0",
  schema: {
    type: "object",
    properties: {
      case_number: { type: "string", pattern: "^[0-9]{2}-CV-[0-9]{5}$" },
      filing_date: { type: "string", format: "date" },
      document_type: { type: "string", enum: ["complaint", "motion", "answer"] },
      parties: {
        type: "array",
        items: {
          type: "object",
          properties: {
            name: { type: "string" },
            role: { type: "string", enum: ["plaintiff", "defendant"] }
          }
        }
      }
    },
    required: ["case_number", "filing_date", "document_type"]
  },
  description: "Federal court filing schema (District Court)",
  tags: ["legal", "court", "filing"]
});
```

**Compliance**: E-discovery requires standardized metadata schemas.

---

### 4. Research (RSRCH - Utilitario)

**Use Case**: Manage founder/company fact schemas for RSRCH (utilitario research system).

**Schemas Managed**:
- Fact metadata (subject_entity, claim, fact_type)
- Source credibility schemas
- Entity relationship schemas
- Investment fact schemas
- Founder profile schemas

**Example**:
```typescript
// Register founder fact schema
await registry.publish({
  schema_id: "founder_fact",
  version: "1.0.0",
  schema: {
    type: "object",
    properties: {
      fact_id: { type: "string" },
      fact_type: { type: "string", enum: ["investment", "founding", "employment", "education"] },
      claim: { type: "string" },
      subject_entity: { type: "string" },  // e.g., "Sam Altman"
      discovered_at: { type: "string", format: "date-time" },
      sources: {
        type: "array",
        items: {
          type: "object",
          properties: {
            source_type: { type: "string", enum: ["web_article", "podcast", "tweet", "interview"] },
            source_url: { type: "string" },
            source_credibility: { type: "number", minimum: 0, maximum: 1 }
          }
        }
      }
    },
    required: ["fact_id", "fact_type", "claim", "subject_entity", "sources"]
  },
  description: "Founder fact schema for RSRCH utilitario research",
  tags: ["rsrch", "founder", "fact", "investment"]
});

// Evolve schema: add structured subject_entity object (minor version)
await registry.publish({
  schema_id: "founder_fact",
  version: "1.1.0",
  schema: {
    type: "object",
    properties: {
      fact_id: { type: "string" },
      fact_type: { type: "string", enum: ["investment", "founding", "employment", "education"] },
      claim: { type: "string" },
      subject_entity: {  // Changed from string to object
        type: "object",
        properties: {
          entity_id: { type: "string" },
          entity_name: { type: "string" },
          entity_type: { type: "string", enum: ["founder", "company", "investor"] }
        },
        required: ["entity_id", "entity_name", "entity_type"]
      },
      discovered_at: { type: "string", format: "date-time" },
      sources: {
        type: "array",
        items: {
          type: "object",
          properties: {
            source_type: { type: "string", enum: ["web_article", "podcast", "tweet", "interview"] },
            source_url: { type: "string" },
            source_credibility: { type: "number", minimum: 0, maximum: 1 }
          }
        }
      }
    },
    required: ["fact_id", "fact_type", "claim", "subject_entity", "sources"]
  },
  description: "Founder fact schema - structured subject_entity for entity resolution",
  tags: ["rsrch", "founder", "fact", "investment"]
});
```

---

### 5. E-commerce

**Use Case**: Manage product catalog schemas across marketplaces and inventory systems.

**Schemas Managed**:
- Product listing schemas (Amazon, eBay, Shopify)
- Order schemas
- Inventory update schemas
- Price history schemas
- Customer review schemas

**Example**:
```typescript
// Register product schema
await registry.publish({
  schema_id: "product_listing",
  version: "1.0.0",
  schema: {
    type: "object",
    properties: {
      sku: { type: "string" },
      name: { type: "string", maxLength: 200 },
      description: { type: "string" },
      price: { type: "number", minimum: 0 },
      currency: { type: "string", enum: ["USD", "EUR", "GBP"] },
      inventory_count: { type: "integer", minimum: 0 },
      category: { type: "string" }
    },
    required: ["sku", "name", "price", "currency"]
  },
  description: "Product listing schema for e-commerce platform",
  tags: ["ecommerce", "product", "catalog"]
});

// Breaking change: add required field (major version)
await registry.publish({
  schema_id: "product_listing",
  version: "2.0.0",
  schema: {
    type: "object",
    properties: {
      sku: { type: "string" },
      name: { type: "string", maxLength: 200 },
      description: { type: "string" },
      price: { type: "number", minimum: 0 },
      currency: { type: "string", enum: ["USD", "EUR", "GBP"] },
      inventory_count: { type: "integer", minimum: 0 },
      category: { type: "string" },
      brand: { type: "string" } // New REQUIRED field
    },
    required: ["sku", "name", "price", "currency", "brand"] // Breaking change
  },
  description: "Product listing schema - brand now required",
  tags: ["ecommerce", "product", "catalog"]
});
```

---

### 6. SaaS

**Use Case**: Manage API request/response schemas for multi-tenant SaaS platforms.

**Schemas Managed**:
- API request schemas (REST, GraphQL)
- Webhook payload schemas
- User profile schemas
- Subscription event schemas
- Analytics event schemas

**Example**:
```typescript
// Register webhook payload schema
await registry.publish({
  schema_id: "subscription_webhook",
  version: "1.0.0",
  schema: {
    type: "object",
    properties: {
      event_id: { type: "string" },
      event_type: { type: "string", enum: ["created", "updated", "cancelled"] },
      timestamp: { type: "string", format: "date-time" },
      subscription: {
        type: "object",
        properties: {
          id: { type: "string" },
          customer_id: { type: "string" },
          plan: { type: "string" },
          status: { type: "string", enum: ["active", "cancelled", "past_due"] }
        },
        required: ["id", "customer_id", "plan", "status"]
      }
    },
    required: ["event_id", "event_type", "timestamp", "subscription"]
  },
  description: "Subscription webhook payload for SaaS platform",
  tags: ["saas", "webhook", "subscription"]
});

// Get latest schema for validation
const latestSchema = await registry.getLatest("subscription_webhook");
```

---

### 7. Insurance

**Use Case**: Manage insurance policy and claim schemas across underwriting and claims systems.

**Schemas Managed**:
- Policy application schemas
- Quote schemas
- Claim submission schemas
- Adjudication result schemas
- Payout schemas

**Example**:
```typescript
// Register auto insurance claim schema
await registry.publish({
  schema_id: "auto_insurance_claim",
  version: "1.0.0",
  schema: {
    type: "object",
    properties: {
      claim_id: { type: "string" },
      policy_number: { type: "string" },
      incident_date: { type: "string", format: "date" },
      description: { type: "string" },
      estimated_damage: { type: "number", minimum: 0 },
      claimant: {
        type: "object",
        properties: {
          name: { type: "string" },
          phone: { type: "string" }
        },
        required: ["name", "phone"]
      }
    },
    required: ["claim_id", "policy_number", "incident_date", "claimant"]
  },
  description: "Auto insurance claim submission schema",
  tags: ["insurance", "claim", "auto"]
});
```

---

### Cross-Domain Benefits

**Consistent Data Contracts**: All domains benefit from:
- Single source of truth for schema definitions
- Semantic versioning prevents breaking changes
- Backward compatibility checking before deployment
- Complete history of schema evolution

**Regulatory Compliance**: Supports:
- SOX (Finance): Schema versioning for data integrity
- HIPAA (Healthcare): Data structure validation
- GDPR (Privacy): Data lineage tracking
- ASC 606 (Revenue): Schema audit trail

---

## Core Concepts

### Schema Definition

A **schema** is a JSON Schema document that defines the structure, types, and validation rules for data.

**Example**:
```json
{
  "type": "object",
  "properties": {
    "id": { "type": "string" },
    "name": { "type": "string" },
    "age": { "type": "integer", "minimum": 0 }
  },
  "required": ["id", "name"]
}
```

### Schema Identity

Every schema has a unique identifier:
- **schema_id**: Logical name (e.g., "bank_transaction", "patient_record")
- **version**: Semantic version (e.g., "1.2.3")
- **UNIQUE constraint**: (schema_id, version) enforced at database level

### Semantic Versioning

Follows [semver](https://semver.org/) conventions:

- **MAJOR**: Breaking changes (remove field, change type, add required field)
- **MINOR**: Backward-compatible additions (add optional field, relax constraint)
- **PATCH**: Bug fixes (fix description, correct typo, update example)

**Examples**:
```typescript
// PATCH: v1.0.0 → v1.0.1 (fix typo in description)
// MINOR: v1.0.1 → v1.1.0 (add optional field)
// MAJOR: v1.1.0 → v2.0.0 (remove field)
```

### Schema States

- **DRAFT**: Under development, not published
- **PUBLISHED**: Live and available for use
- **DEPRECATED**: Superseded by newer version, still available
- **ARCHIVED**: No longer supported, read-only

### Schema Metadata

Every schema version includes:
- `schema_id`: Unique identifier
- `version`: Semantic version
- `schema`: JSON Schema definition (JSONB)
- `description`: Human-readable description
- `tags`: Array of tags for search/discovery
- `published_at`: Timestamp when published
- `published_by`: User who published
- `status`: Current status (DRAFT, PUBLISHED, DEPRECATED, ARCHIVED)

---

## Interface Definition

### TypeScript Interface

```typescript
interface SchemaRegistry {
  /**
   * Publish a new schema version
   *
   * @param schemaVersion - Schema definition with metadata
   * @returns Published schema version record
   * @throws ValidationError if schema is invalid
   * @throws ConflictError if version already exists
   */
  publish(schemaVersion: SchemaVersionInput): Promise<SchemaVersion>;

  /**
   * Get the latest version of a schema
   *
   * @param schemaId - Schema identifier
   * @returns Latest published schema version, or null if not found
   */
  getLatest(schemaId: string): Promise<SchemaVersion | null>;

  /**
   * Get a specific version of a schema
   *
   * @param schemaId - Schema identifier
   * @param version - Semantic version (e.g., "1.2.3")
   * @returns Schema version, or null if not found
   */
  getVersion(schemaId: string, version: string): Promise<SchemaVersion | null>;

  /**
   * List all versions of a schema
   *
   * @param schemaId - Schema identifier
   * @param options - Optional filters (status, sort order)
   * @returns Array of schema versions in descending version order
   */
  listVersions(
    schemaId: string,
    options?: ListVersionsOptions
  ): Promise<SchemaVersion[]>;

  /**
   * Search schemas by name, description, tags, or field names
   *
   * @param query - Search query parameters
   * @returns Array of matching schema versions
   */
  searchSchemas(query: SearchQuery): Promise<SchemaVersion[]>;

  /**
   * Deprecate a schema version
   *
   * @param schemaId - Schema identifier
   * @param version - Version to deprecate
   * @param reason - Deprecation reason
   * @returns Updated schema version
   */
  deprecate(
    schemaId: string,
    version: string,
    reason: string
  ): Promise<SchemaVersion>;

  /**
   * Archive a schema version (mark as no longer supported)
   *
   * @param schemaId - Schema identifier
   * @param version - Version to archive
   * @returns Updated schema version
   */
  archive(schemaId: string, version: string): Promise<SchemaVersion>;

  /**
   * Get schema statistics
   *
   * @param schemaId - Schema identifier
   * @returns Usage and version statistics
   */
  getStats(schemaId: string): Promise<SchemaStats>;

  /**
   * List all schemas in registry
   *
   * @param options - Pagination and filtering options
   * @returns Array of schema metadata (latest version only)
   */
  listSchemas(options?: ListSchemasOptions): Promise<SchemaSummary[]>;

  /**
   * Validate data against a schema version
   *
   * @param schemaId - Schema identifier
   * @param version - Schema version
   * @param data - Data to validate
   * @returns Validation result with errors (if any)
   */
  validate(
    schemaId: string,
    version: string,
    data: any
  ): Promise<ValidationResult>;
}
```

---

## Data Model

### SchemaVersion Type

```typescript
interface SchemaVersion {
  // Identity
  schema_id: string;              // Logical schema name (e.g., "bank_transaction")
  version: string;                // Semantic version (e.g., "1.2.3")

  // Schema Definition
  schema: object;                 // JSON Schema definition (stored as JSONB)

  // Metadata
  description: string;            // Human-readable description
  tags: string[];                 // Tags for search/discovery
  published_at: string;           // ISO timestamp
  published_by: string;           // User ID who published
  status: SchemaStatus;           // DRAFT | PUBLISHED | DEPRECATED | ARCHIVED

  // Deprecation
  deprecated_at?: string;         // ISO timestamp (if deprecated)
  deprecation_reason?: string;    // Reason for deprecation

  // System
  created_at: string;             // ISO timestamp
  updated_at: string;             // ISO timestamp
}
```

### SchemaVersionInput Type

```typescript
interface SchemaVersionInput {
  schema_id: string;
  version: string;
  schema: object;                 // JSON Schema definition
  description: string;
  tags?: string[];
  status?: SchemaStatus;          // Default: PUBLISHED
}
```

### SearchQuery Type

```typescript
interface SearchQuery {
  // Full-text search
  query?: string;                 // Search in name, description, field names

  // Filters
  tags?: string[];                // Filter by tags
  status?: SchemaStatus[];        // Filter by status
  published_after?: string;       // Filter by publish date

  // Pagination
  limit?: number;                 // Default: 50
  offset?: number;                // Default: 0

  // Sorting
  sort_by?: 'published_at' | 'schema_id' | 'version';
  sort_order?: 'asc' | 'desc';    // Default: 'desc'
}
```

### SchemaStats Type

```typescript
interface SchemaStats {
  schema_id: string;
  total_versions: number;
  latest_version: string;
  published_versions: number;
  deprecated_versions: number;
  first_published: string;        // ISO timestamp
  last_published: string;         // ISO timestamp
}
```

### Database Schema (PostgreSQL)

```sql
CREATE TABLE schema_registry (
  -- Primary Key
  schema_id VARCHAR(128) NOT NULL,
  version VARCHAR(32) NOT NULL,
  PRIMARY KEY (schema_id, version),

  -- Schema Definition
  schema JSONB NOT NULL,

  -- Metadata
  description TEXT NOT NULL,
  tags TEXT[] DEFAULT '{}',
  published_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  published_by VARCHAR(128) NOT NULL,
  status VARCHAR(32) NOT NULL DEFAULT 'PUBLISHED',

  -- Deprecation
  deprecated_at TIMESTAMP WITH TIME ZONE,
  deprecation_reason TEXT,

  -- System
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Indexes for Performance
CREATE INDEX idx_schema_registry_schema_id ON schema_registry(schema_id);
CREATE INDEX idx_schema_registry_status ON schema_registry(status);
CREATE INDEX idx_schema_registry_tags ON schema_registry USING GIN(tags);
CREATE INDEX idx_schema_registry_published_at ON schema_registry(published_at DESC);

-- Full-text search index on description
CREATE INDEX idx_schema_registry_description_fts ON schema_registry USING GIN(to_tsvector('english', description));

-- GIN index for JSONB schema field queries
CREATE INDEX idx_schema_registry_schema ON schema_registry USING GIN(schema);

-- UNIQUE constraint enforced by PRIMARY KEY
-- ALTER TABLE schema_registry ADD CONSTRAINT uq_schema_version UNIQUE (schema_id, version);

-- Trigger to update updated_at
CREATE OR REPLACE FUNCTION update_schema_registry_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER schema_registry_updated_at
BEFORE UPDATE ON schema_registry
FOR EACH ROW
EXECUTE FUNCTION update_schema_registry_updated_at();
```

---

## Core Functionality

### 1. publish()

Publish a new schema version to the registry.

#### Signature

```typescript
publish(schemaVersion: SchemaVersionInput): Promise<SchemaVersion>
```

#### Behavior

1. **Validation**:
   - Validate schema against JSON Schema meta-schema
   - Validate semantic version format (major.minor.patch)
   - Check if version already exists (UNIQUE constraint)

2. **Compatibility Check** (optional):
   - If previous version exists, run backward compatibility check
   - Warn if breaking changes detected without major version bump

3. **Insert**:
   - INSERT into `schema_registry` table
   - Return published schema version

4. **Error Handling**:
   - Throw `ValidationError` if schema is invalid
   - Throw `ConflictError` if version already exists
   - Throw `CompatibilityError` if breaking changes in non-major version

#### Example

```typescript
const schemaVersion = await registry.publish({
  schema_id: "user_profile",
  version: "1.0.0",
  schema: {
    type: "object",
    properties: {
      id: { type: "string" },
      email: { type: "string", format: "email" },
      name: { type: "string" }
    },
    required: ["id", "email"]
  },
  description: "User profile schema for authentication service",
  tags: ["user", "authentication"]
});

console.log(`Published ${schemaVersion.schema_id} v${schemaVersion.version}`);
```

#### Error Cases

```typescript
// Case 1: Invalid JSON Schema
try {
  await registry.publish({
    schema_id: "invalid_schema",
    version: "1.0.0",
    schema: {
      type: "invalid_type" // Not a valid JSON Schema type
    },
    description: "Invalid schema"
  });
} catch (error) {
  console.log(error.code); // "VALIDATION_ERROR"
  console.log(error.message); // "Invalid JSON Schema: type must be one of [object, array, string, number, boolean, null]"
}

// Case 2: Version already exists
try {
  await registry.publish({
    schema_id: "user_profile",
    version: "1.0.0", // Already exists
    schema: { type: "object" },
    description: "Duplicate version"
  });
} catch (error) {
  console.log(error.code); // "CONFLICT_ERROR"
  console.log(error.message); // "Version 1.0.0 already exists for schema user_profile"
}

// Case 3: Invalid semantic version
try {
  await registry.publish({
    schema_id: "user_profile",
    version: "1.0", // Invalid semver (missing patch)
    schema: { type: "object" },
    description: "Invalid version"
  });
} catch (error) {
  console.log(error.code); // "VALIDATION_ERROR"
  console.log(error.message); // "Version must follow semver format (major.minor.patch)"
}
```

---

### 2. getLatest()

Get the latest published version of a schema.

#### Signature

```typescript
getLatest(schemaId: string): Promise<SchemaVersion | null>
```

#### Behavior

1. Query `schema_registry` WHERE `schema_id = $1` AND `status = 'PUBLISHED'`
2. Sort by `version DESC` (using semantic version comparison)
3. Return first row (latest version), or null if not found

#### Example

```typescript
const latestSchema = await registry.getLatest("user_profile");

if (latestSchema) {
  console.log(`Latest version: ${latestSchema.version}`);
  console.log(`Published at: ${latestSchema.published_at}`);
  console.log(`Schema:`, latestSchema.schema);
} else {
  console.log("Schema not found");
}
```

#### Performance

- **Target Latency**: < 10ms (indexed query)
- **Index Used**: `idx_schema_registry_schema_id` + sorting

---

### 3. getVersion()

Get a specific version of a schema.

#### Signature

```typescript
getVersion(schemaId: string, version: string): Promise<SchemaVersion | null>
```

#### Behavior

1. Query `schema_registry` WHERE `schema_id = $1` AND `version = $2`
2. Return row, or null if not found

#### Example

```typescript
// Get specific version
const schema_v1 = await registry.getVersion("user_profile", "1.0.0");

if (schema_v1) {
  console.log(`Version 1.0.0 schema:`, schema_v1.schema);
} else {
  console.log("Version not found");
}

// Compare with latest
const latestSchema = await registry.getLatest("user_profile");

if (latestSchema && latestSchema.version !== "1.0.0") {
  console.log(`New version available: ${latestSchema.version}`);
}
```

#### Performance

- **Target Latency**: < 5ms (PRIMARY KEY lookup)
- **Index Used**: PRIMARY KEY (schema_id, version)

---

### 4. listVersions()

List all versions of a schema.

#### Signature

```typescript
listVersions(
  schemaId: string,
  options?: ListVersionsOptions
): Promise<SchemaVersion[]>
```

#### Behavior

1. Query `schema_registry` WHERE `schema_id = $1`
2. Apply filters (status, etc.)
3. Sort by `version DESC` (semantic version)
4. Return array of schema versions

#### Example

```typescript
// Get all versions
const allVersions = await registry.listVersions("user_profile");

console.log(`Total versions: ${allVersions.length}`);

for (const version of allVersions) {
  console.log(`v${version.version} - ${version.status} - ${version.published_at}`);
}

// Output:
// v2.0.0 - PUBLISHED - 2025-03-01T10:00:00Z
// v1.1.0 - DEPRECATED - 2025-02-01T10:00:00Z
// v1.0.0 - DEPRECATED - 2025-01-01T10:00:00Z

// Get only published versions
const publishedVersions = await registry.listVersions("user_profile", {
  status: ["PUBLISHED"]
});

console.log(`Published versions: ${publishedVersions.length}`);
```

---

### 5. searchSchemas()

Search schemas by name, description, tags, or field names.

#### Signature

```typescript
searchSchemas(query: SearchQuery): Promise<SchemaVersion[]>
```

#### Behavior

1. Build full-text search query
2. Apply filters (tags, status, date)
3. Execute query with LIMIT/OFFSET
4. Return array of matching schema versions

#### Example

```typescript
// Search by query string
const results = await registry.searchSchemas({
  query: "transaction",
  limit: 10
});

console.log(`Found ${results.length} schemas matching "transaction"`);

for (const schema of results) {
  console.log(`${schema.schema_id} v${schema.version}: ${schema.description}`);
}

// Search by tags
const financialSchemas = await registry.searchSchemas({
  tags: ["finance", "banking"]
});

console.log(`Found ${financialSchemas.length} financial schemas`);

// Search with filters
const recentSchemas = await registry.searchSchemas({
  status: ["PUBLISHED"],
  published_after: "2025-01-01T00:00:00Z",
  sort_by: "published_at",
  sort_order: "desc",
  limit: 20
});
```

#### Full-Text Search

Uses PostgreSQL full-text search on `description` field:

```sql
SELECT * FROM schema_registry
WHERE to_tsvector('english', description) @@ plainto_tsquery('english', $1)
ORDER BY published_at DESC
LIMIT $2 OFFSET $3;
```

---

### 6. deprecate()

Mark a schema version as deprecated.

#### Signature

```typescript
deprecate(
  schemaId: string,
  version: string,
  reason: string
): Promise<SchemaVersion>
```

#### Behavior

1. Fetch schema version
2. Update `status = 'DEPRECATED'`
3. Set `deprecated_at = NOW()`
4. Set `deprecation_reason = reason`
5. Return updated schema version

#### Example

```typescript
// Deprecate old version
await registry.deprecate(
  "user_profile",
  "1.0.0",
  "Superseded by v2.0.0 with improved validation"
);

const deprecated = await registry.getVersion("user_profile", "1.0.0");

console.log(`Status: ${deprecated.status}`); // "DEPRECATED"
console.log(`Deprecated at: ${deprecated.deprecated_at}`);
console.log(`Reason: ${deprecated.deprecation_reason}`);
```

---

### 7. validate()

Validate data against a schema version.

#### Signature

```typescript
validate(
  schemaId: string,
  version: string,
  data: any
): Promise<ValidationResult>
```

#### Behavior

1. Fetch schema version
2. Validate `data` against JSON Schema
3. Return validation result with errors (if any)

#### Example

```typescript
// Valid data
const result1 = await registry.validate(
  "user_profile",
  "1.0.0",
  {
    id: "user_123",
    email: "alice@example.com",
    name: "Alice"
  }
);

console.log(result1.valid); // true
console.log(result1.errors); // []

// Invalid data (missing required field)
const result2 = await registry.validate(
  "user_profile",
  "1.0.0",
  {
    id: "user_123",
    // missing email (required)
    name: "Bob"
  }
);

console.log(result2.valid); // false
console.log(result2.errors); // [{ field: 'email', message: 'is required' }]
```

---

## Versioning Strategy

### Semantic Versioning Rules

Following [semver](https://semver.org/):

**MAJOR (x.0.0)**: Breaking changes
- Remove field
- Rename field
- Change field type
- Add required field
- Tighten constraint (e.g., increase minimum)

**MINOR (x.y.0)**: Backward-compatible additions
- Add optional field
- Relax constraint (e.g., decrease minimum)
- Add enum value

**PATCH (x.y.z)**: Bug fixes and non-functional changes
- Fix description typo
- Update documentation
- Correct example

### Version Comparison

```typescript
function compareVersions(v1: string, v2: string): number {
  const [major1, minor1, patch1] = v1.split('.').map(Number);
  const [major2, minor2, patch2] = v2.split('.').map(Number);

  if (major1 !== major2) return major1 - major2;
  if (minor1 !== minor2) return minor1 - minor2;
  return patch1 - patch2;
}

// Usage
console.log(compareVersions("1.0.0", "2.0.0")); // -1 (v1 < v2)
console.log(compareVersions("2.1.0", "2.0.0")); // 1 (v1 > v2)
console.log(compareVersions("1.0.0", "1.0.0")); // 0 (v1 == v2)
```

### Version Selection Strategy

**For Producers** (writing data):
- Always use latest PUBLISHED version
- Check for deprecation warnings

**For Consumers** (reading data):
- Support all non-ARCHIVED versions
- Prioritize latest version
- Handle missing optional fields gracefully

---

## Schema Evolution

### Evolution Patterns

#### Pattern 1: Add Optional Field (MINOR)

```typescript
// v1.0.0
{
  type: "object",
  properties: {
    id: { type: "string" },
    name: { type: "string" }
  },
  required: ["id", "name"]
}

// v1.1.0: Add optional field
{
  type: "object",
  properties: {
    id: { type: "string" },
    name: { type: "string" },
    email: { type: "string", format: "email" } // New optional field
  },
  required: ["id", "name"]
}
```

#### Pattern 2: Add Required Field (MAJOR)

```typescript
// v1.0.0
{
  type: "object",
  properties: {
    id: { type: "string" },
    name: { type: "string" }
  },
  required: ["id", "name"]
}

// v2.0.0: Add required field (BREAKING)
{
  type: "object",
  properties: {
    id: { type: "string" },
    name: { type: "string" },
    email: { type: "string", format: "email" } // New REQUIRED field
  },
  required: ["id", "name", "email"] // BREAKING: email now required
}
```

#### Pattern 3: Change Field Type (MAJOR)

```typescript
// v1.0.0
{
  type: "object",
  properties: {
    id: { type: "string" },
    age: { type: "string" } // Age as string
  },
  required: ["id"]
}

// v2.0.0: Change type (BREAKING)
{
  type: "object",
  properties: {
    id: { type: "string" },
    age: { type: "integer" } // BREAKING: type changed from string to integer
  },
  required: ["id"]
}
```

---

## Storage & Indexing

### PostgreSQL JSONB Storage

Schema definitions stored as JSONB for:
- Efficient querying
- Indexing support (GIN indexes)
- JSON validation

**Query Schema Fields**:
```sql
-- Find schemas with specific field
SELECT schema_id, version, description
FROM schema_registry
WHERE schema->'properties' ? 'email';

-- Find schemas with required field
SELECT schema_id, version
FROM schema_registry
WHERE schema->'required' @> '["email"]'::jsonb;
```

### Index Strategy

```sql
-- B-tree indexes for exact matches
CREATE INDEX idx_schema_registry_schema_id ON schema_registry(schema_id);
CREATE INDEX idx_schema_registry_status ON schema_registry(status);

-- GIN indexes for array/JSONB queries
CREATE INDEX idx_schema_registry_tags ON schema_registry USING GIN(tags);
CREATE INDEX idx_schema_registry_schema ON schema_registry USING GIN(schema);

-- Full-text search index
CREATE INDEX idx_schema_registry_description_fts ON schema_registry USING GIN(to_tsvector('english', description));

-- Time-based index for recent queries
CREATE INDEX idx_schema_registry_published_at ON schema_registry(published_at DESC);
```

---

## Edge Cases

### Edge Case 1: Version Already Exists

**Scenario**: Attempt to publish a version that already exists.

**Handling**: Return `ConflictError` with existing version details:

```typescript
try {
  await registry.publish({
    schema_id: "user_profile",
    version: "1.0.0", // Already exists
    schema: { type: "object" },
    description: "Duplicate"
  });
} catch (error) {
  if (error.code === 'CONFLICT_ERROR') {
    console.log(`Version already exists: ${error.existingVersion.published_at}`);
    console.log(`Published by: ${error.existingVersion.published_by}`);
  }
}
```

**Database Guarantee**: PRIMARY KEY (schema_id, version) prevents duplicates.

---

### Edge Case 2: Invalid Semantic Version

**Scenario**: Version string doesn't follow semver format.

**Handling**: Validate format before insertion:

```typescript
function isValidSemver(version: string): boolean {
  const semverRegex = /^(\d+)\.(\d+)\.(\d+)$/;
  return semverRegex.test(version);
}

// Usage
if (!isValidSemver("1.0")) {
  throw new ValidationError("Version must follow semver format (major.minor.patch)");
}
```

---

### Edge Case 3: No Published Versions

**Scenario**: Schema has only DRAFT or DEPRECATED versions.

**Handling**: `getLatest()` returns `null`:

```typescript
const latest = await registry.getLatest("draft_only_schema");

if (latest === null) {
  console.log("No published versions available");

  // Optionally check for draft versions
  const allVersions = await registry.listVersions("draft_only_schema");
  const drafts = allVersions.filter(v => v.status === 'DRAFT');

  console.log(`Found ${drafts.length} draft versions`);
}
```

---

### Edge Case 4: Circular Schema References

**Scenario**: Schema references itself or creates circular dependency.

**Handling**: JSON Schema supports `$ref`, but circular references must be handled carefully:

```typescript
// Valid: Schema with $ref
{
  "type": "object",
  "properties": {
    "parent": { "$ref": "#/definitions/Person" }
  },
  "definitions": {
    "Person": {
      "type": "object",
      "properties": {
        "name": { "type": "string" }
      }
    }
  }
}

// Problematic: Circular reference across schemas
// Schema A references Schema B, which references Schema A
// Recommendation: Use explicit versioning in $ref
{
  "$ref": "schema_registry://user_profile/1.0.0#/definitions/User"
}
```

---

### Edge Case 5: Very Large Schema Definitions

**Scenario**: Schema definition exceeds reasonable size (e.g., > 1MB).

**Handling**: Set size limit and recommend splitting:

```typescript
const MAX_SCHEMA_SIZE = 1 * 1024 * 1024; // 1MB

function validateSchemaSize(schema: object): void {
  const size = Buffer.byteLength(JSON.stringify(schema), 'utf8');

  if (size > MAX_SCHEMA_SIZE) {
    throw new ValidationError(
      `Schema size (${size} bytes) exceeds maximum (${MAX_SCHEMA_SIZE} bytes). ` +
      `Consider splitting into multiple schemas with $ref.`
    );
  }
}
```

---

### Edge Case 6: Breaking Change in MINOR Version

**Scenario**: Developer accidentally introduces breaking change in minor version.

**Handling**: Integration with `BackwardCompatibilityChecker` to detect and prevent:

```typescript
async function publishWithCompatibilityCheck(
  input: SchemaVersionInput
): Promise<SchemaVersion> {
  // Get previous version
  const versions = await registry.listVersions(input.schema_id);

  if (versions.length > 0) {
    const latestVersion = versions[0];

    // Check compatibility
    const checker = new BackwardCompatibilityChecker();
    const report = await checker.check(latestVersion.schema, input.schema);

    // Detect version type from semver
    const [newMajor] = input.version.split('.').map(Number);
    const [oldMajor] = latestVersion.version.split('.').map(Number);

    const isMajorBump = newMajor > oldMajor;

    // If breaking changes detected and not major bump, error
    if (report.breaking_changes.length > 0 && !isMajorBump) {
      throw new CompatibilityError(
        `Breaking changes detected but version is not major bump. ` +
        `Breaking changes: ${report.breaking_changes.map(c => c.description).join(', ')}`
      );
    }
  }

  return registry.publish(input);
}
```

---

### Edge Case 7: Deprecate Latest Version

**Scenario**: Latest version is deprecated, what should `getLatest()` return?

**Handling**: `getLatest()` only returns PUBLISHED versions:

```typescript
// Scenario: v2.0.0 exists but is DEPRECATED
await registry.deprecate("user_profile", "2.0.0", "Found critical bug");

// getLatest() returns previous published version
const latest = await registry.getLatest("user_profile");
console.log(latest.version); // "1.1.0" (previous published version)

// To get all versions including deprecated:
const allVersions = await registry.listVersions("user_profile");
```

---

## Performance Characteristics

### Latency Targets

| Operation | Input Size | Target Latency (p95) | Notes |
|-----------|-----------|---------------------|-------|
| `publish()` | 1 schema | < 20ms | Single INSERT with validation |
| `getLatest()` | 1 schema_id | < 10ms | Indexed query + version sort |
| `getVersion()` | 1 (schema_id, version) | < 5ms | PRIMARY KEY lookup |
| `listVersions()` | 1 schema_id | < 20ms | Indexed query + sort |
| `searchSchemas()` | Full-text query | < 100ms | Full-text search + filters |
| `deprecate()` | 1 version | < 15ms | Single UPDATE |
| `validate()` | 1 schema + data | < 50ms | JSON Schema validation |

### Throughput

- **Writes**: 500-1,000 publishes/sec (single node)
- **Reads**: 10,000-50,000 queries/sec (with indexes)

### Scalability

| Total Schemas | Query Performance | Notes |
|--------------|------------------|-------|
| < 1K | Excellent (< 10ms) | All queries fast |
| 1K - 10K | Excellent (< 20ms) | Indexed queries fast |
| 10K - 100K | Good (< 50ms) | Full-text search slower |
| > 100K | Moderate (< 200ms) | Consider sharding by domain |

### Optimization Tips

**1. Cache Latest Versions**

```typescript
class CachedSchemaRegistry implements SchemaRegistry {
  private cache = new Map<string, SchemaVersion>();

  async getLatest(schemaId: string): Promise<SchemaVersion | null> {
    // Check cache first
    if (this.cache.has(schemaId)) {
      return this.cache.get(schemaId);
    }

    // Cache miss: query database
    const schema = await this.registry.getLatest(schemaId);

    if (schema) {
      this.cache.set(schemaId, schema);
    }

    return schema;
  }

  async publish(input: SchemaVersionInput): Promise<SchemaVersion> {
    const schema = await this.registry.publish(input);

    // Invalidate cache for this schema_id
    this.cache.delete(input.schema_id);

    return schema;
  }
}
```

**2. Batch Version Fetches**

```typescript
// Bad: N individual queries
for (const schemaId of schemaIds) {
  const schema = await registry.getLatest(schemaId); // N round trips
}

// Good: Batch query
const schemas = await registry.batchGetLatest(schemaIds); // 1 round trip
```

**3. Limit Search Results**

```typescript
// Bad: Fetch all results
const results = await registry.searchSchemas({ query: "transaction" });

// Good: Limit results with pagination
const results = await registry.searchSchemas({
  query: "transaction",
  limit: 20,
  offset: 0
});
```

---

## Implementation Notes

### PostgreSQL Implementation

```typescript
import { Pool } from 'pg';
import Ajv from 'ajv';
import addFormats from 'ajv-formats';

export class PostgresSchemaRegistry implements SchemaRegistry {
  private pool: Pool;
  private ajv: Ajv;

  constructor(pool: Pool) {
    this.pool = pool;

    // Initialize JSON Schema validator
    this.ajv = new Ajv({ allErrors: true });
    addFormats(this.ajv);
  }

  async publish(input: SchemaVersionInput): Promise<SchemaVersion> {
    // Validate semver format
    if (!this.isValidSemver(input.version)) {
      throw new ValidationError("Version must follow semver format (major.minor.patch)");
    }

    // Validate JSON Schema
    try {
      this.ajv.compile(input.schema);
    } catch (error) {
      throw new ValidationError(`Invalid JSON Schema: ${error.message}`);
    }

    // Insert
    const query = `
      INSERT INTO schema_registry (
        schema_id, version, schema, description, tags, published_by, status
      ) VALUES ($1, $2, $3, $4, $5, $6, $7)
      RETURNING *
    `;

    const values = [
      input.schema_id,
      input.version,
      JSON.stringify(input.schema),
      input.description,
      input.tags || [],
      input.published_by || 'system',
      input.status || 'PUBLISHED'
    ];

    try {
      const result = await this.pool.query(query, values);
      return this.rowToSchemaVersion(result.rows[0]);
    } catch (error) {
      if (error.code === '23505') { // Unique violation
        throw new ConflictError(`Version ${input.version} already exists for schema ${input.schema_id}`);
      }
      throw error;
    }
  }

  async getLatest(schemaId: string): Promise<SchemaVersion | null> {
    const query = `
      SELECT * FROM schema_registry
      WHERE schema_id = $1 AND status = 'PUBLISHED'
      ORDER BY
        split_part(version, '.', 1)::int DESC,
        split_part(version, '.', 2)::int DESC,
        split_part(version, '.', 3)::int DESC
      LIMIT 1
    `;

    const result = await this.pool.query(query, [schemaId]);

    if (result.rows.length === 0) {
      return null;
    }

    return this.rowToSchemaVersion(result.rows[0]);
  }

  async getVersion(schemaId: string, version: string): Promise<SchemaVersion | null> {
    const query = `
      SELECT * FROM schema_registry
      WHERE schema_id = $1 AND version = $2
    `;

    const result = await this.pool.query(query, [schemaId, version]);

    if (result.rows.length === 0) {
      return null;
    }

    return this.rowToSchemaVersion(result.rows[0]);
  }

  async listVersions(
    schemaId: string,
    options?: ListVersionsOptions
  ): Promise<SchemaVersion[]> {
    let query = `
      SELECT * FROM schema_registry
      WHERE schema_id = $1
    `;
    const values: any[] = [schemaId];

    if (options?.status) {
      query += ` AND status = ANY($2)`;
      values.push(options.status);
    }

    query += `
      ORDER BY
        split_part(version, '.', 1)::int DESC,
        split_part(version, '.', 2)::int DESC,
        split_part(version, '.', 3)::int DESC
    `;

    const result = await this.pool.query(query, values);
    return result.rows.map(row => this.rowToSchemaVersion(row));
  }

  async searchSchemas(query: SearchQuery): Promise<SchemaVersion[]> {
    let sql = `SELECT * FROM schema_registry WHERE 1=1`;
    const values: any[] = [];
    let paramCount = 0;

    // Full-text search
    if (query.query) {
      paramCount++;
      sql += ` AND to_tsvector('english', description) @@ plainto_tsquery('english', $${paramCount})`;
      values.push(query.query);
    }

    // Filter by tags
    if (query.tags && query.tags.length > 0) {
      paramCount++;
      sql += ` AND tags && $${paramCount}`;
      values.push(query.tags);
    }

    // Filter by status
    if (query.status && query.status.length > 0) {
      paramCount++;
      sql += ` AND status = ANY($${paramCount})`;
      values.push(query.status);
    }

    // Sort
    const sortBy = query.sort_by || 'published_at';
    const sortOrder = query.sort_order || 'desc';
    sql += ` ORDER BY ${sortBy} ${sortOrder.toUpperCase()}`;

    // Pagination
    const limit = query.limit || 50;
    const offset = query.offset || 0;
    paramCount++;
    sql += ` LIMIT $${paramCount}`;
    values.push(limit);
    paramCount++;
    sql += ` OFFSET $${paramCount}`;
    values.push(offset);

    const result = await this.pool.query(sql, values);
    return result.rows.map(row => this.rowToSchemaVersion(row));
  }

  async deprecate(
    schemaId: string,
    version: string,
    reason: string
  ): Promise<SchemaVersion> {
    const query = `
      UPDATE schema_registry
      SET status = 'DEPRECATED',
          deprecated_at = NOW(),
          deprecation_reason = $3
      WHERE schema_id = $1 AND version = $2
      RETURNING *
    `;

    const result = await this.pool.query(query, [schemaId, version, reason]);

    if (result.rows.length === 0) {
      throw new NotFoundError(`Schema ${schemaId} version ${version} not found`);
    }

    return this.rowToSchemaVersion(result.rows[0]);
  }

  async validate(
    schemaId: string,
    version: string,
    data: any
  ): Promise<ValidationResult> {
    const schemaVersion = await this.getVersion(schemaId, version);

    if (!schemaVersion) {
      throw new NotFoundError(`Schema ${schemaId} version ${version} not found`);
    }

    const validate = this.ajv.compile(schemaVersion.schema);
    const valid = validate(data);

    return {
      valid,
      errors: valid ? [] : (validate.errors || []).map(err => ({
        field: err.instancePath || err.schemaPath,
        message: err.message || 'validation failed'
      }))
    };
  }

  private isValidSemver(version: string): boolean {
    const semverRegex = /^(\d+)\.(\d+)\.(\d+)$/;
    return semverRegex.test(version);
  }

  private rowToSchemaVersion(row: any): SchemaVersion {
    return {
      schema_id: row.schema_id,
      version: row.version,
      schema: typeof row.schema === 'string' ? JSON.parse(row.schema) : row.schema,
      description: row.description,
      tags: row.tags,
      published_at: row.published_at,
      published_by: row.published_by,
      status: row.status,
      deprecated_at: row.deprecated_at,
      deprecation_reason: row.deprecation_reason,
      created_at: row.created_at,
      updated_at: row.updated_at
    };
  }
}
```

---

## Security Considerations

### 1. Schema Validation

**Prevent Malicious Schemas**:
```typescript
// Reject schemas with excessive complexity
function validateSchemaComplexity(schema: object): void {
  const maxDepth = 10;
  const maxProperties = 100;

  function checkDepth(obj: any, depth: number): void {
    if (depth > maxDepth) {
      throw new ValidationError(`Schema depth exceeds maximum (${maxDepth})`);
    }

    if (obj.properties && Object.keys(obj.properties).length > maxProperties) {
      throw new ValidationError(`Schema properties exceed maximum (${maxProperties})`);
    }

    if (obj.properties) {
      for (const prop of Object.values(obj.properties)) {
        checkDepth(prop, depth + 1);
      }
    }
  }

  checkDepth(schema, 0);
}
```

### 2. Access Control

**Role-Based Publishing**:
```typescript
async function publishWithAuth(
  input: SchemaVersionInput,
  user: User
): Promise<SchemaVersion> {
  // Check if user has permission to publish schemas
  if (!user.hasRole('schema_publisher')) {
    throw new UnauthorizedError('User does not have permission to publish schemas');
  }

  // Set published_by to authenticated user
  input.published_by = user.id;

  return registry.publish(input);
}
```

### 3. Immutability

**Prevent Schema Modification**:
```sql
-- Revoke UPDATE on critical fields
CREATE POLICY schema_immutability ON schema_registry
  FOR UPDATE
  USING (false); -- Prevent all updates

-- Allow only status changes (deprecation)
CREATE POLICY schema_status_update ON schema_registry
  FOR UPDATE
  USING (true)
  WITH CHECK (
    status IN ('DEPRECATED', 'ARCHIVED') AND
    schema = (SELECT schema FROM schema_registry WHERE schema_id = schema_registry.schema_id AND version = schema_registry.version)
  );
```

---

## Integration Patterns

### Pattern 1: Schema-First Development

```typescript
// 1. Publish schema
await registry.publish({
  schema_id: "user_created_event",
  version: "1.0.0",
  schema: {
    type: "object",
    properties: {
      user_id: { type: "string" },
      email: { type: "string", format: "email" },
      created_at: { type: "string", format: "date-time" }
    },
    required: ["user_id", "email", "created_at"]
  },
  description: "User created event for event bus",
  tags: ["event", "user"]
});

// 2. Generate TypeScript types from schema
const schema = await registry.getLatest("user_created_event");
const types = generateTypesFromSchema(schema.schema);

// 3. Validate data before publishing to event bus
const eventData = {
  user_id: "user_123",
  email: "alice@example.com",
  created_at: new Date().toISOString()
};

const validationResult = await registry.validate("user_created_event", "1.0.0", eventData);

if (!validationResult.valid) {
  throw new Error(`Invalid event data: ${validationResult.errors}`);
}

await eventBus.publish("user.created", eventData);
```

### Pattern 2: Automatic Schema Discovery

```typescript
class SchemaAwareAPI {
  async handleRequest(req: Request): Promise<Response> {
    // Get schema for endpoint
    const schemaId = `api_${req.path}_${req.method}`.toLowerCase();
    const schema = await registry.getLatest(schemaId);

    if (!schema) {
      return { status: 404, body: { error: "Endpoint schema not found" } };
    }

    // Validate request body
    const validation = await registry.validate(schemaId, schema.version, req.body);

    if (!validation.valid) {
      return {
        status: 400,
        body: {
          error: "Invalid request",
          errors: validation.errors
        }
      };
    }

    // Process request
    return this.processRequest(req);
  }
}
```

---

## Multi-Domain Examples

(Already covered in "Multi-Domain Applicability" section with 7 domains: Finance, Healthcare, Legal, Research, E-commerce, SaaS, Insurance)

---

## Testing Strategy

### Unit Tests

```typescript
describe('SchemaRegistry', () => {
  let registry: SchemaRegistry;
  let pool: Pool;

  beforeEach(async () => {
    pool = new Pool({ /* config */ });
    registry = new PostgresSchemaRegistry(pool);
    await cleanDatabase();
  });

  describe('publish()', () => {
    it('should publish valid schema', async () => {
      const schema = await registry.publish({
        schema_id: 'test_schema',
        version: '1.0.0',
        schema: {
          type: 'object',
          properties: {
            id: { type: 'string' }
          },
          required: ['id']
        },
        description: 'Test schema',
        tags: ['test']
      });

      expect(schema.schema_id).toBe('test_schema');
      expect(schema.version).toBe('1.0.0');
      expect(schema.status).toBe('PUBLISHED');
    });

    it('should reject invalid semver', async () => {
      await expect(
        registry.publish({
          schema_id: 'test_schema',
          version: '1.0', // Invalid
          schema: { type: 'object' },
          description: 'Test'
        })
      ).rejects.toThrow('Version must follow semver format');
    });

    it('should reject duplicate version', async () => {
      await registry.publish({
        schema_id: 'test_schema',
        version: '1.0.0',
        schema: { type: 'object' },
        description: 'Test'
      });

      await expect(
        registry.publish({
          schema_id: 'test_schema',
          version: '1.0.0', // Duplicate
          schema: { type: 'object' },
          description: 'Test 2'
        })
      ).rejects.toThrow('Version 1.0.0 already exists');
    });
  });

  describe('getLatest()', () => {
    it('should return latest published version', async () => {
      await registry.publish({
        schema_id: 'test_schema',
        version: '1.0.0',
        schema: { type: 'object' },
        description: 'v1'
      });

      await registry.publish({
        schema_id: 'test_schema',
        version: '1.1.0',
        schema: { type: 'object' },
        description: 'v1.1'
      });

      const latest = await registry.getLatest('test_schema');
      expect(latest?.version).toBe('1.1.0');
    });

    it('should skip deprecated versions', async () => {
      await registry.publish({
        schema_id: 'test_schema',
        version: '1.0.0',
        schema: { type: 'object' },
        description: 'v1'
      });

      await registry.publish({
        schema_id: 'test_schema',
        version: '2.0.0',
        schema: { type: 'object' },
        description: 'v2'
      });

      await registry.deprecate('test_schema', '2.0.0', 'Bug found');

      const latest = await registry.getLatest('test_schema');
      expect(latest?.version).toBe('1.0.0'); // Falls back to v1
    });
  });

  describe('validate()', () => {
    it('should validate data against schema', async () => {
      await registry.publish({
        schema_id: 'user',
        version: '1.0.0',
        schema: {
          type: 'object',
          properties: {
            id: { type: 'string' },
            email: { type: 'string', format: 'email' }
          },
          required: ['id', 'email']
        },
        description: 'User schema'
      });

      const result = await registry.validate('user', '1.0.0', {
        id: 'user_123',
        email: 'alice@example.com'
      });

      expect(result.valid).toBe(true);
      expect(result.errors).toEqual([]);
    });

    it('should return errors for invalid data', async () => {
      await registry.publish({
        schema_id: 'user',
        version: '1.0.0',
        schema: {
          type: 'object',
          properties: {
            id: { type: 'string' },
            email: { type: 'string', format: 'email' }
          },
          required: ['id', 'email']
        },
        description: 'User schema'
      });

      const result = await registry.validate('user', '1.0.0', {
        id: 'user_123'
        // missing email
      });

      expect(result.valid).toBe(false);
      expect(result.errors.length).toBeGreaterThan(0);
    });
  });
});
```

---

## Migration Guide

### From Hardcoded Schemas to Registry

```typescript
// Before: Hardcoded schema in code
const USER_SCHEMA = {
  type: "object",
  properties: {
    id: { type: "string" },
    email: { type: "string" }
  },
  required: ["id", "email"]
};

function validateUser(data: any): boolean {
  const validate = ajv.compile(USER_SCHEMA);
  return validate(data);
}

// After: Schema from registry
async function validateUser(data: any): Promise<boolean> {
  const schema = await registry.getLatest("user");

  if (!schema) {
    throw new Error("User schema not found");
  }

  const result = await registry.validate("user", schema.version, data);
  return result.valid;
}
```

---

## Simplicity Profiles

### Personal Profile (20 LOC)

**Contexto del Usuario:**
Aplicación personal con schema fijo (transaction: date, amount, merchant, category). Schema hardcoded como TypeScript interface, sin validación runtime.

**Implementation:**
```typescript
// types.ts (Personal - 20 LOC)
interface Transaction {
  id: string;
  date: Date;
  amount: number;
  merchant: string;
  category: string;
}

// No validation - TypeScript compiler checks types at build time
function processTransaction(tx: Transaction) {
  // Business logic here
}
```

**Características Incluidas:**
- ✅ Type safety en compile-time (TypeScript)
- ✅ Schema como código (interfaces)

**Características NO Incluidas:**
- ❌ Runtime validation (YAGNI: trusted data sources)
- ❌ Versioning (YAGNI: schema nunca cambia)
- ❌ JSON Schema files (TypeScript interfaces suficientes)

**Configuración:**
```typescript
// No config needed - schema in code
```

**Performance:**
- Validation: 0ms (compile-time only)

**Upgrade Triggers:**
- Si schema cambia >2 veces/año → Small Business (JSON Schema files)
- Si necesita validación runtime → Small Business (JSON Schema validation)

---

### Small Business Profile (90 LOC)

**Contexto del Usuario:**
Firma con API REST que recibe uploads de 10 clientes. Necesitan validar payloads contra JSON Schema para evitar datos malformados.

**Implementation:**
```python
# schema_registry.py (Small Business - 90 LOC)
import json
from jsonschema import validate, ValidationError

# Schema files stored in schemas/ directory
SCHEMAS = {
    "transaction": "schemas/transaction.v1.json",
    "upload": "schemas/upload.v1.json",
}

def load_schema(schema_name: str):
    """Load JSON Schema from file."""
    schema_path = SCHEMAS[schema_name]
    with open(schema_path, 'r') as f:
        return json.load(f)

def validate_payload(schema_name: str, payload: dict):
    """
    Validate payload against JSON Schema.
    Raises ValidationError if invalid.
    """
    schema = load_schema(schema_name)
    validate(instance=payload, schema=schema)  # Raises if invalid

# Example usage
try:
    payload = {"date": "2025-01-15", "amount": 42.50, "merchant": "Starbucks"}
    validate_payload("transaction", payload)
    print("✓ Valid payload")
except ValidationError as e:
    print(f"✗ Invalid payload: {e.message}")
```

**JSON Schema example:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Transaction",
  "type": "object",
  "required": ["date", "amount", "merchant"],
  "properties": {
    "date": {"type": "string", "format": "date"},
    "amount": {"type": "number"},
    "merchant": {"type": "string", "minLength": 1},
    "category": {"type": "string"}
  }
}
```

**Características Incluidas:**
- ✅ JSON Schema validation (runtime)
- ✅ Schema files (versioned en Git)
- ✅ Clear error messages (ValidationError)

**Características NO Incluidas:**
- ❌ Semantic versioning (solo v1, v2 manualmente)
- ❌ Backward compatibility checks (manual review)
- ❌ Schema registry server (archivos locales)

**Configuración:**
```yaml
schema_registry:
  schemas_dir: "./schemas"
  schemas:
    transaction: "transaction.v1.json"
    upload: "upload.v1.json"
```

**Performance:**
- Validation: <1ms (JSON Schema validación)

**Upgrade Triggers:**
- Si >10 schemas → Enterprise (registry server)
- Si necesita versionado semántico → Enterprise (v1.0.0 format)
- Si >3 services usan schemas → Enterprise (centralized registry)

---

### Enterprise Profile (450 LOC)

**Contexto del Usuario:**
FinTech con 50+ microservices, 100+ schemas, semantic versioning estricto (v1.2.3), backward compatibility checks automáticos, schema registry HTTP server.

**Implementation:**
```python
# schema_registry.py (Enterprise - 450 LOC)
from fastapi import FastAPI, HTTPException
import psycopg2
from typing import Optional
from dataclasses import dataclass

app = FastAPI()

@dataclass
class Schema:
    schema_id: str
    version: str  # Semantic versioning: v1.2.3
    schema_json: dict
    created_at: str
    deprecated: bool

class SchemaRegistry:
    def __init__(self, db):
        self.db = db

    def register_schema(self, schema_id: str, version: str, schema_json: dict) -> Schema:
        """
        Register new schema version.

        Steps:
        1. Parse semantic version (v1.2.3)
        2. Check backward compatibility with latest version
        3. Insert into database (UNIQUE constraint on schema_id + version)
        4. Return registered schema
        """
        # 1. Validate semantic version format
        if not self._is_valid_semver(version):
            raise ValueError(f"Invalid semver: {version}")

        # 2. Check backward compatibility
        latest = self.get_latest_version(schema_id)
        if latest:
            if not self._is_backward_compatible(latest.schema_json, schema_json):
                raise ValueError(f"Breaking change detected in {schema_id} {version}")

        # 3. Insert into database
        self.db.execute("""
            INSERT INTO schemas (schema_id, version, schema_json, created_at)
            VALUES (%s, %s, %s, NOW())
        """, (schema_id, version, json.dumps(schema_json)))

        return Schema(schema_id, version, schema_json, created_at=..., deprecated=False)

    def get_schema(self, schema_id: str, version: Optional[str] = None) -> Schema:
        """
        Get schema by ID and version.
        If version=None, returns latest version.
        """
        if version is None:
            # Get latest version
            row = self.db.execute("""
                SELECT schema_id, version, schema_json, created_at, deprecated
                FROM schemas
                WHERE schema_id = %s
                ORDER BY version DESC
                LIMIT 1
            """, (schema_id,)).fetchone()
        else:
            # Get specific version
            row = self.db.execute("""
                SELECT schema_id, version, schema_json, created_at, deprecated
                FROM schemas
                WHERE schema_id = %s AND version = %s
            """, (schema_id, version)).fetchone()

        if not row:
            raise ValueError(f"Schema not found: {schema_id} {version}")

        return Schema(row[0], row[1], json.loads(row[2]), row[3], row[4])

    def _is_backward_compatible(self, old_schema: dict, new_schema: dict) -> bool:
        """
        Check if new schema is backward compatible with old schema.

        Rules:
        - Can add optional fields (backward compatible)
        - Cannot remove required fields (breaking change)
        - Cannot change field types (breaking change)
        """
        old_required = set(old_schema.get("required", []))
        new_required = set(new_schema.get("required", []))

        # Check if required fields were removed (breaking)
        removed_required = old_required - new_required
        if removed_required:
            return False  # Breaking change

        # Check if field types changed (breaking)
        old_props = old_schema.get("properties", {})
        new_props = new_schema.get("properties", {})

        for field in old_required:
            if field in old_props and field in new_props:
                if old_props[field]["type"] != new_props[field]["type"]:
                    return False  # Type change = breaking

        return True  # Backward compatible

# HTTP API
registry = SchemaRegistry(db)

@app.post("/schemas/{schema_id}")
async def register_schema(schema_id: str, version: str, schema: dict):
    """Register new schema version."""
    try:
        result = registry.register_schema(schema_id, version, schema)
        return {"schema_id": result.schema_id, "version": result.version}
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@app.get("/schemas/{schema_id}")
async def get_schema(schema_id: str, version: Optional[str] = None):
    """Get schema (latest or specific version)."""
    try:
        schema = registry.get_schema(schema_id, version)
        return schema.schema_json
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))
```

**Características Incluidas:**
- ✅ Semantic versioning (v1.2.3 format)
- ✅ Backward compatibility checks (automático)
- ✅ PostgreSQL storage (UNIQUE constraint)
- ✅ HTTP API (FastAPI)
- ✅ Version history (query all versions)

**Características NO Incluidas:**
- ❌ Schema evolution UI (API-only)
- ❌ Migration generation (separado en MigrationEngine)

**Configuración:**
```yaml
schema_registry:
  server:
    host: "0.0.0.0"
    port: 8081
  database:
    url: "postgresql://user:pass@host:5432/schemas"
  compatibility:
    default_mode: "backward"  # backward, forward, full, none
```

**Performance:**
- Latency: <10ms (database query)
- Storage: ~1KB per schema version

**No Further Tiers:**
- Scale horizontally (read replicas)

---

## Related Primitives

- **SchemaVersionManager**: Manages semantic versioning logic and version comparison
- **MigrationEngine**: Generates and executes data migrations between schema versions
- **BackwardCompatibilityChecker**: Detects breaking changes between schema versions
- **ParserRegistry**: Manages parser versions (similar pattern for parser selection)

---

## References

- [JSON Schema Specification](https://json-schema.org/)
- [Semantic Versioning (semver)](https://semver.org/)
- [Objective Layer Architecture](../architecture/objective-layer.md)
- [Schema Registry Vertical 5.2](../verticals/schema-registry.md)
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)

---

**End of SchemaRegistry OL Primitive Specification**
