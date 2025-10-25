# PrecedenceEngine

**Primitive Type:** Objective Layer (OL)
**Domain:** Corrections Flow (Vertical 4.3)
**Status:** Core Primitive
**Version:** 1.0.0

## Table of Contents

1. [Overview](#overview)
2. [Purpose and Scope](#purpose-and-scope)
3. [Architecture](#architecture)
4. [Core Interface](#core-interface)
5. [Precedence Rules](#precedence-rules)
6. [Implementation Patterns](#implementation-patterns)
7. [Multi-Domain Examples](#multi-domain-examples)
8. [Edge Cases and Error Handling](#edge-cases-and-error-handling)
9. [Performance Characteristics](#performance-characteristics)
10. [Integration Points](#integration-points)
11. [Testing Strategy](#testing-strategy)
12. [Migration Guide](#migration-guide)
13. [Related Primitives](#related-primitives)
14. [Appendix](#appendix)

---

## Overview

The **PrecedenceEngine** is a stateless resolution service that determines final field values by applying a fixed precedence hierarchy across multiple value sources. It serves as the authoritative decision-making component in the Corrections Flow, ensuring consistent field resolution across all entity types and domains.

### Key Characteristics

- **Stateless**: Each resolution is independent; no internal caching or state retention
- **Deterministic**: Same inputs always produce the same output
- **Performance-Optimized**: Batch operations complete in <100ms for 100 entities
- **Domain-Agnostic**: Works with any entity type and field schema
- **Audit-Ready**: Every resolution includes complete metadata about sources and decisions

### Design Philosophy

The PrecedenceEngine embodies a simple but powerful principle: **explicit overrides always win, then rules, then observations, then defaults**. This hierarchy reflects the natural authority chain in data quality management:

1. **Manual Override**: Human judgment (highest authority)
2. **Automated Rule**: System-defined business logic
3. **Extraction**: Raw observed data (parser/OCR output)
4. **Default**: Fallback when no other source exists

This design eliminates ambiguity in field resolution and provides a clear audit trail for every value decision.

---

## Purpose and Scope

### Primary Purpose

The PrecedenceEngine resolves the final value for entity fields by applying precedence rules to multiple value sources. It acts as the single source of truth for "what value should this field have right now?"

### Scope

**In Scope:**
- Single-field resolution with complete metadata
- Batch resolution for multiple fields across multiple entities
- Precedence rule retrieval and documentation
- Performance-optimized bulk operations
- Override detection and metadata generation

**Out of Scope:**
- Value storage (handled by EntityStorage)
- Override persistence (handled by CorrectionTracker)
- Rule definition and management (handled by RuleEngine)
- Value extraction (handled by Parsers/OCR)
- User permissions and access control

### Use Cases

1. **Real-Time Display**: Generate display values for UI (e.g., transaction detail view)
2. **Export Generation**: Resolve all fields before exporting to external systems
3. **Reporting**: Ensure reports use correct precedence-resolved values
4. **Validation**: Verify that displayed values match precedence logic
5. **Audit**: Provide transparency about why a field has a specific value
6. **Migration**: Bulk-resolve values when importing legacy data

---

## Architecture

### System Context

```
┌─────────────────────────────────────────────────────────────┐
│                      Objective Layer                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │ EntityStorage│───▶│PrecedenceEng │◀───│CorrectionTrkr│ │
│  └──────────────┘    └──────────────┘    └──────────────┘ │
│                             │                               │
│                             ▼                               │
│                      ┌──────────────┐                      │
│                      │  RuleEngine  │                      │
│                      └──────────────┘                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌──────────────────┐
                    │ Representation   │
                    │ Layer (Output)   │
                    └──────────────────┘
```

### Data Flow

```
Input: Entity ID + Field Name + Value Sources
  │
  ├─▶ 1. Check Manual Override (highest priority)
  │      └─▶ If exists: return with override metadata
  │
  ├─▶ 2. Check Rule Value
  │      └─▶ If exists: return with rule metadata
  │
  ├─▶ 3. Check Extraction Value
  │      └─▶ If exists: return with extraction metadata
  │
  └─▶ 4. Check Default Value
         └─▶ If exists: return with default metadata
         └─▶ If null: return null with metadata

Output: PrecedenceResolution (value + source + metadata)
```

### Component Relationships

**Upstream Dependencies:**
- **EntityStorage**: Provides extraction values from parsed documents
- **CorrectionTracker**: Provides manual override values
- **RuleEngine**: Provides rule-generated values

**Downstream Consumers:**
- **Representation Layer**: Uses resolved values for display
- **Export Services**: Uses resolved values for external systems
- **Reporting Services**: Uses resolved values for analytics
- **Validation Services**: Verifies resolution logic correctness

---

## Core Interface

### TypeScript Interface

```typescript
/**
 * PrecedenceEngine resolves final field values using a fixed precedence hierarchy.
 *
 * Precedence Order (highest to lowest):
 * 1. Manual Override
 * 2. Automated Rule
 * 3. Extraction (parser/OCR)
 * 4. Default
 */
interface PrecedenceEngine {
  /**
   * Resolve a single field's final value.
   *
   * @param entityId - Unique identifier for the entity
   * @param fieldName - Name of the field to resolve
   * @param sources - Available value sources for this field
   * @returns Resolution containing final value and metadata
   *
   * @example
   * const resolution = await engine.resolveField(
   *   'tx_12345',
   *   'merchant',
   *   {
   *     manual_override: 'Amazon Marketplace',
   *     extraction_value: 'AMZN MKTP US',
   *     default_value: 'Unknown Merchant'
   *   }
   * );
   * // resolution.final_value === 'Amazon Marketplace'
   * // resolution.source === 'manual'
   */
  resolveField(
    entityId: string,
    fieldName: string,
    sources: ValueSources
  ): Promise<PrecedenceResolution>;

  /**
   * Resolve all fields for a single entity.
   *
   * @param entityId - Unique identifier for the entity
   * @param fieldsWithSources - Map of field names to their value sources
   * @returns Map of field names to their resolutions
   *
   * @example
   * const resolutions = await engine.resolveAll('tx_12345', {
   *   merchant: { manual_override: 'Amazon', extraction_value: 'AMZN' },
   *   amount: { extraction_value: 49.99, default_value: 0 }
   * });
   */
  resolveAll(
    entityId: string,
    fieldsWithSources: Record<string, ValueSources>
  ): Promise<Record<string, PrecedenceResolution>>;

  /**
   * Resolve all fields for multiple entities (batch operation).
   *
   * @param batch - Map of entity IDs to their field sources
   * @returns Map of entity IDs to their field resolutions
   *
   * @performance <100ms for 100 entities with 10 fields each
   *
   * @example
   * const batchResolutions = await engine.resolveAllBatch({
   *   'tx_001': { merchant: {...}, amount: {...} },
   *   'tx_002': { merchant: {...}, amount: {...} }
   * });
   */
  resolveAllBatch(
    batch: Record<string, Record<string, ValueSources>>
  ): Promise<Map<string, Record<string, PrecedenceResolution>>>;

  /**
   * Get the precedence rules used by this engine.
   *
   * @returns Array of precedence rules in priority order
   */
  getPrecedenceRules(): PrecedenceRule[];

  /**
   * Explain why a field resolved to its current value.
   *
   * @param entityId - Entity identifier
   * @param fieldName - Field to explain
   * @param sources - Value sources
   * @returns Human-readable explanation
   *
   * @example
   * const explanation = await engine.explainResolution(
   *   'tx_12345',
   *   'merchant',
   *   sources
   * );
   * // "Field 'merchant' resolved to 'Amazon Marketplace' because a manual
   * //  override was set by user@example.com on 2025-10-15T10:30:00Z,
   * //  overriding extraction value 'AMZN MKTP US'."
   */
  explainResolution(
    entityId: string,
    fieldName: string,
    sources: ValueSources
  ): Promise<string>;
}

/**
 * Value sources for a single field.
 * All sources are optional; precedence determines which is used.
 */
interface ValueSources {
  /** Manual override set by a user (highest priority) */
  manual_override?: any;

  /** Value generated by automated rule */
  rule_value?: any;

  /** Value extracted from document (parser/OCR) */
  extraction_value?: any;

  /** Default/fallback value (lowest priority) */
  default_value?: any;
}

/**
 * Result of field resolution with complete metadata.
 */
interface PrecedenceResolution {
  /** Name of the resolved field */
  field_name: string;

  /** Final resolved value (null if no sources available) */
  final_value: any;

  /** Source that provided the final value */
  source: 'manual' | 'rule' | 'extraction' | 'default' | null;

  /** All available sources (for audit trail) */
  sources: ValueSources;

  /** Additional metadata about the resolution */
  metadata: {
    /** Whether a manual override is active */
    is_overridden: boolean;

    /** User who created the override */
    overridden_by?: string;

    /** Timestamp of override creation */
    overridden_at?: string;

    /** Values that were skipped (lower priority) */
    skipped_sources?: Array<{
      source: 'rule' | 'extraction' | 'default';
      value: any;
    }>;

    /** Confidence score (if applicable) */
    confidence?: number;
  };
}

/**
 * Precedence rule definition.
 */
interface PrecedenceRule {
  /** Priority level (1 = highest) */
  priority: number;

  /** Source name */
  source: 'manual' | 'rule' | 'extraction' | 'default';

  /** Human-readable description */
  description: string;

  /** When this rule applies */
  conditions?: string[];
}
```

### Python Implementation Signature

```python
from typing import Any, Dict, List, Optional
from dataclasses import dataclass
from datetime import datetime

@dataclass
class ValueSources:
    """Available value sources for field resolution."""
    manual_override: Optional[Any] = None
    rule_value: Optional[Any] = None
    extraction_value: Optional[Any] = None
    default_value: Optional[Any] = None

@dataclass
class SkippedSource:
    """A value source that was skipped during resolution."""
    source: str  # 'rule' | 'extraction' | 'default'
    value: Any

@dataclass
class ResolutionMetadata:
    """Metadata about field resolution."""
    is_overridden: bool
    overridden_by: Optional[str] = None
    overridden_at: Optional[datetime] = None
    skipped_sources: Optional[List[SkippedSource]] = None
    confidence: Optional[float] = None

@dataclass
class PrecedenceResolution:
    """Result of field resolution."""
    field_name: str
    final_value: Any
    source: Optional[str]  # 'manual' | 'rule' | 'extraction' | 'default' | None
    sources: ValueSources
    metadata: ResolutionMetadata

@dataclass
class PrecedenceRule:
    """Precedence rule definition."""
    priority: int
    source: str  # 'manual' | 'rule' | 'extraction' | 'default'
    description: str
    conditions: Optional[List[str]] = None

class PrecedenceEngine:
    """
    Stateless field value resolution using precedence rules.
    """

    async def resolve_field(
        self,
        entity_id: str,
        field_name: str,
        sources: ValueSources
    ) -> PrecedenceResolution:
        """Resolve a single field's final value."""
        pass

    async def resolve_all(
        self,
        entity_id: str,
        fields_with_sources: Dict[str, ValueSources]
    ) -> Dict[str, PrecedenceResolution]:
        """Resolve all fields for a single entity."""
        pass

    async def resolve_all_batch(
        self,
        batch: Dict[str, Dict[str, ValueSources]]
    ) -> Dict[str, Dict[str, PrecedenceResolution]]:
        """Resolve all fields for multiple entities."""
        pass

    def get_precedence_rules(self) -> List[PrecedenceRule]:
        """Get the precedence rules used by this engine."""
        pass

    async def explain_resolution(
        self,
        entity_id: str,
        field_name: str,
        sources: ValueSources
    ) -> str:
        """Explain why a field resolved to its current value."""
        pass
```

---

## Precedence Rules

### Rule Hierarchy

The PrecedenceEngine uses a **static, universal hierarchy** that applies to all fields and domains:

```
Priority 1: MANUAL OVERRIDE
  ├─ Description: User explicitly set this value
  ├─ Authority: Human judgment
  ├─ When to use: Corrections, clarifications, enrichments
  └─ Metadata: user_id, timestamp, reason (optional)

Priority 2: AUTOMATED RULE
  ├─ Description: System-generated value from business rules
  ├─ Authority: Rule engine / business logic
  ├─ When to use: Normalization, categorization, calculations
  └─ Metadata: rule_id, rule_version, confidence

Priority 3: EXTRACTION
  ├─ Description: Value extracted from source document
  ├─ Authority: Parser / OCR / data source
  ├─ When to use: Raw data from documents, APIs, files
  └─ Metadata: parser_version, confidence, location

Priority 4: DEFAULT
  ├─ Description: Fallback value when no other source exists
  ├─ Authority: Schema definition
  ├─ When to use: Schema defaults, null handling
  └─ Metadata: schema_version, default_reason
```

### Rule Application Logic

```typescript
function resolveField(sources: ValueSources): any {
  // Check sources in priority order
  if (sources.manual_override !== undefined && sources.manual_override !== null) {
    return {
      value: sources.manual_override,
      source: 'manual'
    };
  }

  if (sources.rule_value !== undefined && sources.rule_value !== null) {
    return {
      value: sources.rule_value,
      source: 'rule'
    };
  }

  if (sources.extraction_value !== undefined && sources.extraction_value !== null) {
    return {
      value: sources.extraction_value,
      source: 'extraction'
    };
  }

  if (sources.default_value !== undefined && sources.default_value !== null) {
    return {
      value: sources.default_value,
      source: 'default'
    };
  }

  // No sources available
  return {
    value: null,
    source: null
  };
}
```

### Override Detection

A field is considered "overridden" when:

1. A manual override exists AND
2. At least one lower-priority source exists with a different value

```typescript
function detectOverride(resolution: PrecedenceResolution): boolean {
  if (resolution.source !== 'manual') {
    return false;
  }

  const manualValue = resolution.sources.manual_override;
  const lowerPrioritySources = [
    resolution.sources.rule_value,
    resolution.sources.extraction_value,
    resolution.sources.default_value
  ];

  // Check if any lower-priority source exists with a different value
  for (const lowerValue of lowerPrioritySources) {
    if (lowerValue !== undefined &&
        lowerValue !== null &&
        !deepEqual(lowerValue, manualValue)) {
      return true;
    }
  }

  return false;
}
```

### Null Handling

The PrecedenceEngine treats `null` and `undefined` differently:

- **`null`**: Explicit "no value" (participates in precedence)
- **`undefined`**: Source not available (skipped in precedence)

```typescript
// Example: Explicit null override
const sources = {
  manual_override: null,        // User explicitly set to null
  extraction_value: 'SOME_VALUE'
};

// Resolution: final_value = null, source = 'manual'
// Interpretation: User intentionally cleared this field
```

```typescript
// Example: Missing source
const sources = {
  manual_override: undefined,   // No override exists
  extraction_value: 'SOME_VALUE'
};

// Resolution: final_value = 'SOME_VALUE', source = 'extraction'
// Interpretation: No override, use extraction
```

---

## Implementation Patterns

### Pattern 1: Single Field Resolution

**Use Case**: Display a single field value in UI

```typescript
// Scenario: Display merchant name for a transaction
async function displayMerchant(txId: string): Promise<string> {
  const engine = new PrecedenceEngine();

  // Gather sources from various systems
  const sources: ValueSources = {
    manual_override: await correctionTracker.getOverride(txId, 'merchant'),
    rule_value: await ruleEngine.evaluateField(txId, 'merchant'),
    extraction_value: await entityStorage.getField(txId, 'merchant'),
    default_value: 'Unknown Merchant'
  };

  const resolution = await engine.resolveField(txId, 'merchant', sources);

  return resolution.final_value;
}
```

### Pattern 2: Full Entity Resolution

**Use Case**: Prepare entity for display or export

```typescript
// Scenario: Export transaction to accounting system
async function exportTransaction(txId: string): Promise<ExportedTransaction> {
  const engine = new PrecedenceEngine();
  const fields = ['merchant', 'amount', 'category', 'date', 'description'];

  // Batch gather all sources
  const fieldsWithSources: Record<string, ValueSources> = {};

  for (const field of fields) {
    fieldsWithSources[field] = {
      manual_override: await correctionTracker.getOverride(txId, field),
      rule_value: await ruleEngine.evaluateField(txId, field),
      extraction_value: await entityStorage.getField(txId, field),
      default_value: getSchemaDefault(field)
    };
  }

  const resolutions = await engine.resolveAll(txId, fieldsWithSources);

  // Convert to export format
  return {
    id: txId,
    merchant: resolutions.merchant.final_value,
    amount: resolutions.amount.final_value,
    category: resolutions.category.final_value,
    date: resolutions.date.final_value,
    description: resolutions.description.final_value
  };
}
```

### Pattern 3: Batch Resolution

**Use Case**: Resolve values for multiple entities efficiently

```typescript
// Scenario: Generate monthly report for 1000 transactions
async function generateMonthlyReport(txIds: string[]): Promise<Report> {
  const engine = new PrecedenceEngine();

  // Batch fetch all overrides upfront (1 query)
  const allOverrides = await correctionTracker.getOverridesBatch(txIds);

  // Batch fetch all extractions upfront (1 query)
  const allExtractions = await entityStorage.getEntitiesBatch(txIds);

  // Prepare batch input
  const batch: Record<string, Record<string, ValueSources>> = {};

  for (const txId of txIds) {
    batch[txId] = {
      merchant: {
        manual_override: allOverrides[txId]?.merchant,
        extraction_value: allExtractions[txId]?.merchant,
        default_value: 'Unknown'
      },
      amount: {
        manual_override: allOverrides[txId]?.amount,
        extraction_value: allExtractions[txId]?.amount,
        default_value: 0
      }
      // ... other fields
    };
  }

  // Resolve all in one call
  const resolutions = await engine.resolveAllBatch(batch);

  // Generate report
  return createReport(resolutions);
}
```

### Pattern 4: Audit Trail Generation

**Use Case**: Show why a field has its current value

```typescript
// Scenario: User clicks "Why is this value?" button
async function showValueExplanation(
  entityId: string,
  fieldName: string
): Promise<string> {
  const engine = new PrecedenceEngine();

  const sources: ValueSources = {
    manual_override: await correctionTracker.getOverride(entityId, fieldName),
    rule_value: await ruleEngine.evaluateField(entityId, fieldName),
    extraction_value: await entityStorage.getField(entityId, fieldName),
    default_value: getSchemaDefault(fieldName)
  };

  const explanation = await engine.explainResolution(
    entityId,
    fieldName,
    sources
  );

  return explanation;

  // Example output:
  // "The 'merchant' field shows 'Amazon Marketplace' because:
  //  1. You manually set this value on Oct 15, 2025 at 10:30 AM
  //  2. This overrides the extracted value 'AMZN MKTP US'
  //  3. The automated rule suggested 'Amazon', but your manual entry takes precedence"
}
```

### Pattern 5: Override Detection for UI

**Use Case**: Show visual indicator when field is manually overridden

```typescript
// Scenario: Display transaction with override badges
async function renderTransactionCard(txId: string): Promise<JSX.Element> {
  const engine = new PrecedenceEngine();

  const fieldsWithSources = await gatherSources(txId, [
    'merchant', 'amount', 'category'
  ]);

  const resolutions = await engine.resolveAll(txId, fieldsWithSources);

  return (
    <TransactionCard>
      <Field
        name="Merchant"
        value={resolutions.merchant.final_value}
        isOverridden={resolutions.merchant.metadata.is_overridden}
        overriddenBy={resolutions.merchant.metadata.overridden_by}
      />
      <Field
        name="Amount"
        value={resolutions.amount.final_value}
        isOverridden={resolutions.amount.metadata.is_overridden}
      />
      <Field
        name="Category"
        value={resolutions.category.final_value}
        isOverridden={resolutions.category.metadata.is_overridden}
      />
    </TransactionCard>
  );
}
```

### Pattern 6: Pre-Resolution Optimization

**Use Case**: Avoid unnecessary source fetches when precedence is known

```typescript
// Scenario: If manual override exists, skip lower-priority fetches
async function optimizedResolveField(
  entityId: string,
  fieldName: string
): Promise<any> {
  const engine = new PrecedenceEngine();

  // Check manual override first
  const manualOverride = await correctionTracker.getOverride(entityId, fieldName);

  if (manualOverride !== undefined && manualOverride !== null) {
    // Manual override exists - skip lower-priority sources
    return await engine.resolveField(entityId, fieldName, {
      manual_override: manualOverride
    });
  }

  // Check rule value
  const ruleValue = await ruleEngine.evaluateField(entityId, fieldName);

  if (ruleValue !== undefined && ruleValue !== null) {
    // Rule exists - skip extraction and default
    return await engine.resolveField(entityId, fieldName, {
      rule_value: ruleValue
    });
  }

  // Fall through to extraction and default
  const extractionValue = await entityStorage.getField(entityId, fieldName);
  const defaultValue = getSchemaDefault(fieldName);

  return await engine.resolveField(entityId, fieldName, {
    extraction_value: extractionValue,
    default_value: defaultValue
  });
}
```

---

## Multi-Domain Examples

### Example 1: Finance - Transaction Merchant Resolution

**Scenario**: Resolve merchant name for a credit card transaction

```typescript
// Input Data
const transactionId = 'tx_finance_001';
const sources: ValueSources = {
  manual_override: 'Amazon Marketplace',
  rule_value: 'Amazon',
  extraction_value: 'AMZN MKTP US*AB12C3D4',
  default_value: 'Unknown Merchant'
};

// Resolution
const resolution = await engine.resolveField(
  transactionId,
  'merchant',
  sources
);

// Output
{
  field_name: 'merchant',
  final_value: 'Amazon Marketplace',
  source: 'manual',
  sources: { /* all sources */ },
  metadata: {
    is_overridden: true,
    overridden_by: 'user@example.com',
    overridden_at: '2025-10-15T10:30:00Z',
    skipped_sources: [
      { source: 'rule', value: 'Amazon' },
      { source: 'extraction', value: 'AMZN MKTP US*AB12C3D4' }
    ]
  }
}
```

**Explanation**: User clarified that "AMZN MKTP" specifically refers to Amazon Marketplace (not Amazon Prime, Amazon Fresh, etc.). The rule engine normalized to just "Amazon", but the manual override provides more specificity.

---

### Example 2: Healthcare - Diagnosis Code Resolution

**Scenario**: Resolve ICD-10 diagnosis code for patient encounter

```typescript
// Input Data
const encounterId = 'enc_health_042';
const sources: ValueSources = {
  manual_override: 'E11.65',  // Type 2 diabetes with hyperglycemia
  rule_value: undefined,       // No rule for this scenario
  extraction_value: 'E11.9',   // Type 2 diabetes without complications (from OCR)
  default_value: 'Z00.00'      // General examination without complaint
};

// Resolution
const resolution = await engine.resolveField(
  encounterId,
  'diagnosis_code',
  sources
);

// Output
{
  field_name: 'diagnosis_code',
  final_value: 'E11.65',
  source: 'manual',
  sources: { /* all sources */ },
  metadata: {
    is_overridden: true,
    overridden_by: 'dr.smith@hospital.com',
    overridden_at: '2025-10-20T14:22:00Z',
    skipped_sources: [
      { source: 'extraction', value: 'E11.9' }
    ]
  }
}
```

**Explanation**: OCR extracted "E11.9" (diabetes without complications) from physician notes, but the doctor corrected it to "E11.65" (diabetes with hyperglycemia) after reviewing lab results. Manual override ensures billing accuracy.

---

### Example 3: Legal - Case Number Resolution

**Scenario**: Resolve court case number from filed document

```typescript
// Input Data
const documentId = 'doc_legal_789';
const sources: ValueSources = {
  manual_override: undefined,          // No override needed
  rule_value: undefined,               // No rule applies
  extraction_value: '2025-CV-1234',    // Extracted from PDF header
  default_value: 'CASE-UNKNOWN'
};

// Resolution
const resolution = await engine.resolveField(
  documentId,
  'case_number',
  sources
);

// Output
{
  field_name: 'case_number',
  final_value: '2025-CV-1234',
  source: 'extraction',
  sources: { /* all sources */ },
  metadata: {
    is_overridden: false,
    skipped_sources: []
  }
}
```

**Explanation**: Extraction successfully parsed the case number from the document header. No override or rule needed; extraction value is authoritative.

---

### Example 4: Research - Author Name Resolution

**Scenario**: Resolve author name for academic paper citation

```typescript
// Input Data
const paperId = 'paper_res_456';
const sources: ValueSources = {
  manual_override: 'Smith, John A.',   // User provided full middle initial
  rule_value: 'Smith, J.',             // Rule normalized from extraction
  extraction_value: 'J. Smith',        // Extracted from PDF metadata
  default_value: 'Unknown Author'
};

// Resolution
const resolution = await engine.resolveField(
  paperId,
  'author',
  sources
);

// Output
{
  field_name: 'author',
  final_value: 'Smith, John A.',
  source: 'manual',
  sources: { /* all sources */ },
  metadata: {
    is_overridden: true,
    overridden_by: 'researcher@university.edu',
    overridden_at: '2025-10-18T09:15:00Z',
    skipped_sources: [
      { source: 'rule', value: 'Smith, J.' },
      { source: 'extraction', value: 'J. Smith' }
    ]
  }
}
```

**Explanation**: PDF metadata had abbreviated author name "J. Smith". Rule engine normalized to "Smith, J." (last name first). Researcher manually corrected to full name "Smith, John A." for citation accuracy.

---

### Example 5: E-commerce - Product Category Resolution

**Scenario**: Resolve category for product listing

```typescript
// Input Data
const productId = 'prod_ecom_991';
const sources: ValueSources = {
  manual_override: 'Home Appliances > Small Kitchen Appliances > Coffee Makers',
  rule_value: 'Electronics > Kitchen Electronics',  // Rule-based categorization
  extraction_value: 'Appliances',                   // Vendor provided
  default_value: 'Uncategorized'
};

// Resolution
const resolution = await engine.resolveField(
  productId,
  'category',
  sources
);

// Output
{
  field_name: 'category',
  final_value: 'Home Appliances > Small Kitchen Appliances > Coffee Makers',
  source: 'manual',
  sources: { /* all sources */ },
  metadata: {
    is_overridden: true,
    overridden_by: 'catalog.admin@store.com',
    overridden_at: '2025-10-19T16:45:00Z',
    skipped_sources: [
      { source: 'rule', value: 'Electronics > Kitchen Electronics' },
      { source: 'extraction', value: 'Appliances' }
    ]
  }
}
```

**Explanation**: Vendor feed provided broad category "Appliances". ML rule classified as "Electronics > Kitchen Electronics". Catalog admin manually placed in specific hierarchy for better product discovery.

---

### Example 6: SaaS - Monthly Recurring Revenue (MRR) Resolution

**Scenario**: Resolve MRR for subscription during pricing correction

```typescript
// Input Data
const subscriptionId = 'sub_saas_123';
const sources: ValueSources = {
  manual_override: 500.00,    // Finance corrected pricing error
  rule_value: undefined,      // No rule override
  extraction_value: 50.00,    // Extracted from payment processor (wrong tier)
  default_value: 0
};

// Resolution
const resolution = await engine.resolveField(
  subscriptionId,
  'mrr',
  sources
);

// Output
{
  field_name: 'mrr',
  final_value: 500.00,
  source: 'manual',
  sources: { /* all sources */ },
  metadata: {
    is_overridden: true,
    overridden_by: 'finance@saascompany.com',
    overridden_at: '2025-10-21T11:20:00Z',
    skipped_sources: [
      { source: 'extraction', value: 50.00 }
    ],
    confidence: 1.0
  }
}
```

**Explanation**: Payment processor reported $50/month (starter tier), but customer was actually on enterprise tier ($500/month) due to manual contract. Finance team overrode to correct MRR for accurate revenue reporting.

---

### Example 7: Insurance - Policy Premium Resolution

**Scenario**: Resolve monthly premium after underwriting adjustment

```typescript
// Input Data
const policyId = 'pol_ins_567';
const sources: ValueSources = {
  manual_override: undefined,  // No manual correction needed
  rule_value: 285.00,          // Adjusted for claims history
  extraction_value: 250.00,    // Initial quote from application
  default_value: 300.00        // Standard rate
};

// Resolution
const resolution = await engine.resolveField(
  policyId,
  'monthly_premium',
  sources
);

// Output
{
  field_name: 'monthly_premium',
  final_value: 285.00,
  source: 'rule',
  sources: { /* all sources */ },
  metadata: {
    is_overridden: false,
    skipped_sources: [
      { source: 'extraction', value: 250.00 }
    ]
  }
}
```

**Explanation**: Initial quote was $250. Underwriting rule adjusted to $285 based on claims history and risk factors. No manual override needed; rule-based pricing is authoritative.

---

## Edge Cases and Error Handling

### Edge Case 1: All Sources Null

**Scenario**: No value available from any source

```typescript
const sources: ValueSources = {
  manual_override: null,
  rule_value: null,
  extraction_value: null,
  default_value: null
};

const resolution = await engine.resolveField('entity_001', 'field_x', sources);

// Output
{
  field_name: 'field_x',
  final_value: null,
  source: 'manual',  // Manual null has highest priority
  sources: { /* all null */ },
  metadata: {
    is_overridden: false  // No override because all sources are null
  }
}
```

**Handling**: Return `null` as final value with source indicating highest-priority non-undefined source. This represents "explicitly no value" rather than "value missing".

---

### Edge Case 2: All Sources Undefined

**Scenario**: No sources provided at all

```typescript
const sources: ValueSources = {
  manual_override: undefined,
  rule_value: undefined,
  extraction_value: undefined,
  default_value: undefined
};

const resolution = await engine.resolveField('entity_002', 'field_y', sources);

// Output
{
  field_name: 'field_y',
  final_value: null,
  source: null,  // No source available
  sources: { /* all undefined */ },
  metadata: {
    is_overridden: false
  }
}
```

**Handling**: Return `null` with `source: null` to indicate no value sources exist. Distinguish from "explicitly null" by checking source.

---

### Edge Case 3: Manual Override Equals Extraction

**Scenario**: User sets manual override to same value as extraction

```typescript
const sources: ValueSources = {
  manual_override: 'AMZN MKTP US',  // User confirmed this is correct
  rule_value: 'Amazon',
  extraction_value: 'AMZN MKTP US',  // Same as manual
  default_value: 'Unknown'
};

const resolution = await engine.resolveField('tx_003', 'merchant', sources);

// Output
{
  field_name: 'merchant',
  final_value: 'AMZN MKTP US',
  source: 'manual',
  sources: { /* all sources */ },
  metadata: {
    is_overridden: true,  // Still marked as override (user confirmed)
    overridden_by: 'user@example.com',
    overridden_at: '2025-10-22T10:00:00Z',
    skipped_sources: [
      { source: 'rule', value: 'Amazon' }  // Only different values listed
    ]
  }
}
```

**Handling**: Even if manual override equals extraction, it's still treated as an override (user explicitly confirmed). Only show different values in `skipped_sources`.

---

### Edge Case 4: Missing Default Value

**Scenario**: Schema doesn't define a default for this field

```typescript
const sources: ValueSources = {
  manual_override: undefined,
  rule_value: undefined,
  extraction_value: undefined,
  default_value: undefined  // No schema default
};

const resolution = await engine.resolveField('entity_004', 'optional_field', sources);

// Output
{
  field_name: 'optional_field',
  final_value: null,
  source: null,
  sources: { /* all undefined */ },
  metadata: {
    is_overridden: false
  }
}
```

**Handling**: Treat as "no value available". Optional fields can legitimately have no default.

---

### Edge Case 5: Type Mismatch Between Sources

**Scenario**: Different sources provide incompatible types

```typescript
const sources: ValueSources = {
  manual_override: '49.99',        // String
  rule_value: undefined,
  extraction_value: 49.99,         // Number
  default_value: 0
};

const resolution = await engine.resolveField('tx_005', 'amount', sources);

// Output
{
  field_name: 'amount',
  final_value: '49.99',  // String from manual override
  source: 'manual',
  sources: { /* all sources */ },
  metadata: {
    is_overridden: true,
    overridden_by: 'user@example.com',
    overridden_at: '2025-10-22T11:30:00Z',
    skipped_sources: [
      { source: 'extraction', value: 49.99 }
    ]
  }
}
```

**Handling**: PrecedenceEngine does NOT perform type coercion. It returns values exactly as provided. Type validation is the responsibility of the caller (Representation Layer, validation services, etc.).

---

### Edge Case 6: Batch Resolution with Partial Failures

**Scenario**: Some entities fail during batch resolution

```typescript
const batch = {
  'entity_001': { field_a: { extraction_value: 'value1' } },
  'entity_002': { field_a: { extraction_value: 'value2' } },
  'entity_003': null  // Invalid input
};

try {
  const resolutions = await engine.resolveAllBatch(batch);
} catch (error) {
  // Error: Invalid input for entity 'entity_003': expected object, got null
}
```

**Handling**: Fail fast on invalid input. Batch operations are transactional at the call level (not database level). If any entity has invalid input, the entire batch fails with descriptive error.

**Alternative Pattern**: Use individual `resolveAll()` calls in a loop if partial success is acceptable.

---

### Edge Case 7: Extremely Large Batch (1000+ Entities)

**Scenario**: Batch resolution for 1000 entities with 20 fields each

```typescript
const largeBatch: Record<string, Record<string, ValueSources>> = {};

for (let i = 0; i < 1000; i++) {
  largeBatch[`entity_${i}`] = {
    field_1: { extraction_value: `value_${i}_1` },
    field_2: { extraction_value: `value_${i}_2` },
    // ... 18 more fields
  };
}

const startTime = Date.now();
const resolutions = await engine.resolveAllBatch(largeBatch);
const duration = Date.now() - startTime;

console.log(`Resolved ${Object.keys(resolutions).length} entities in ${duration}ms`);
// Expected: "Resolved 1000 entities in 75ms"
```

**Handling**: Engine should complete in <100ms for 1000 entities. Implementation should:
1. Use in-memory resolution (no I/O during resolution)
2. Parallelize where possible
3. Avoid unnecessary object cloning
4. Pre-allocate result maps

**Performance Contract**: <100ms for 100 entities, linear scaling up to 1000 entities.

---

### Edge Case 8: Circular Reference in Sources

**Scenario**: Rule value references manual override (circular)

```typescript
// This should NEVER happen in well-designed system, but edge case handling:
const sources: ValueSources = {
  manual_override: getRuleValue(),  // Circular!
  rule_value: getManualOverride(),  // Circular!
  extraction_value: 'safe_value',
  default_value: 'default'
};
```

**Handling**: PrecedenceEngine operates on **pre-computed values only**. It does NOT execute functions or resolve references. The caller must provide fully-resolved values:

```typescript
// Correct usage:
const sources: ValueSources = {
  manual_override: await correctionTracker.getOverride(id, field),  // Resolved before call
  rule_value: await ruleEngine.evaluate(id, field),                // Resolved before call
  extraction_value: await entityStorage.getField(id, field),        // Resolved before call
  default_value: schema.getDefault(field)                           // Resolved before call
};
```

If sources contain functions or promises, engine should throw error:

```typescript
Error: Invalid source value for 'manual_override': expected primitive or object, got function
```

---

## Performance Characteristics

### Latency Targets

| Operation | Target | Typical | Notes |
|-----------|--------|---------|-------|
| `resolveField()` | <1ms | 0.2ms | Single field, in-memory only |
| `resolveAll()` (10 fields) | <5ms | 2ms | Single entity, all fields |
| `resolveAllBatch()` (100 entities) | <100ms | 60ms | Batch operation |
| `resolveAllBatch()` (1000 entities) | <500ms | 300ms | Large batch |
| `explainResolution()` | <10ms | 5ms | Includes string formatting |

### Memory Profile

```
Single Field Resolution: ~500 bytes
  ├─ PrecedenceResolution object: ~300 bytes
  ├─ Metadata object: ~150 bytes
  └─ Overhead: ~50 bytes

Full Entity Resolution (20 fields): ~10 KB
  ├─ 20 × PrecedenceResolution: ~6 KB
  ├─ Result map overhead: ~3 KB
  └─ Working memory: ~1 KB

Batch Resolution (100 entities, 20 fields): ~1 MB
  ├─ 2000 resolutions: ~600 KB
  ├─ Result maps: ~300 KB
  └─ Working memory: ~100 KB
```

### Optimization Strategies

#### Strategy 1: Pre-fetch Sources in Batch

```typescript
// Bad: Fetch sources individually (N queries)
for (const entityId of entityIds) {
  const sources = {
    manual_override: await getOverride(entityId),  // N queries
    extraction_value: await getExtraction(entityId) // N queries
  };
  resolutions[entityId] = await engine.resolveField(entityId, field, sources);
}

// Good: Batch fetch sources upfront (2 queries)
const allOverrides = await getOverridesBatch(entityIds);      // 1 query
const allExtractions = await getExtractionsBatch(entityIds);  // 1 query

const batch = {};
for (const entityId of entityIds) {
  batch[entityId] = {
    field: {
      manual_override: allOverrides[entityId],
      extraction_value: allExtractions[entityId]
    }
  };
}

const resolutions = await engine.resolveAllBatch(batch);  // In-memory only
```

#### Strategy 2: Skip Unnecessary Source Fetches

```typescript
// If manual override exists, skip fetching rule/extraction/default
const override = await getOverride(entityId, field);

if (override !== undefined) {
  // Use override only - save 3 database queries
  return await engine.resolveField(entityId, field, {
    manual_override: override
  });
}

// Otherwise, fetch remaining sources
const rule = await getRule(entityId, field);
if (rule !== undefined) {
  // Use rule only - save 2 database queries
  return await engine.resolveField(entityId, field, {
    rule_value: rule
  });
}

// Fall through to extraction and default
```

#### Strategy 3: Parallel Resolution (for independent entities)

```typescript
// Resolve multiple entities in parallel
const resolutionPromises = entityIds.map(async (entityId) => {
  const sources = await gatherSources(entityId);
  return await engine.resolveAll(entityId, sources);
});

const resolutions = await Promise.all(resolutionPromises);
```

#### Strategy 4: Memoization (for repeated field resolutions)

```typescript
// Cache resolutions within a single request context
class CachedPrecedenceEngine {
  private cache = new Map<string, PrecedenceResolution>();

  async resolveField(
    entityId: string,
    fieldName: string,
    sources: ValueSources
  ): Promise<PrecedenceResolution> {
    const cacheKey = `${entityId}:${fieldName}`;

    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)!;
    }

    const resolution = await this.baseEngine.resolveField(
      entityId,
      fieldName,
      sources
    );

    this.cache.set(cacheKey, resolution);
    return resolution;
  }

  clearCache() {
    this.cache.clear();
  }
}
```

### Scalability Limits

| Scenario | Limit | Mitigation |
|----------|-------|------------|
| Batch size | 1000 entities | Split into multiple batches |
| Fields per entity | 100 fields | Use field projection (only resolve needed fields) |
| Concurrent requests | 1000/sec | Stateless design enables horizontal scaling |
| Memory per request | 10 MB | Batch operations auto-release memory |

---

## Integration Points

### Upstream Systems

#### 1. CorrectionTracker (Manual Overrides)

```typescript
// CorrectionTracker provides manual override values
interface CorrectionTracker {
  getOverride(
    entityId: string,
    fieldName: string
  ): Promise<ManualOverride | undefined>;

  getOverridesBatch(
    entityIds: string[]
  ): Promise<Map<string, Record<string, ManualOverride>>>;
}

interface ManualOverride {
  value: any;
  user_id: string;
  timestamp: string;
  reason?: string;
}

// Integration
async function getManualOverrideSource(
  entityId: string,
  fieldName: string
): Promise<any> {
  const override = await correctionTracker.getOverride(entityId, fieldName);
  return override?.value;  // Return value or undefined
}
```

#### 2. RuleEngine (Automated Rules)

```typescript
// RuleEngine provides rule-generated values
interface RuleEngine {
  evaluateField(
    entityId: string,
    fieldName: string
  ): Promise<RuleResult | undefined>;
}

interface RuleResult {
  value: any;
  rule_id: string;
  confidence: number;
}

// Integration
async function getRuleValueSource(
  entityId: string,
  fieldName: string
): Promise<any> {
  const result = await ruleEngine.evaluateField(entityId, fieldName);
  return result?.value;
}
```

#### 3. EntityStorage (Extraction Values)

```typescript
// EntityStorage provides extracted values from documents
interface EntityStorage {
  getField(
    entityId: string,
    fieldName: string
  ): Promise<any>;

  getEntity(entityId: string): Promise<Record<string, any>>;

  getEntitiesBatch(
    entityIds: string[]
  ): Promise<Map<string, Record<string, any>>>;
}

// Integration
async function getExtractionSource(
  entityId: string,
  fieldName: string
): Promise<any> {
  return await entityStorage.getField(entityId, fieldName);
}
```

#### 4. Schema Registry (Default Values)

```typescript
// Schema provides default values for fields
interface SchemaRegistry {
  getFieldSchema(entityType: string, fieldName: string): FieldSchema;
}

interface FieldSchema {
  name: string;
  type: string;
  required: boolean;
  default?: any;
}

// Integration
function getDefaultValueSource(
  entityType: string,
  fieldName: string
): any {
  const schema = schemaRegistry.getFieldSchema(entityType, fieldName);
  return schema.default;
}
```

### Downstream Consumers

#### 1. Representation Layer

```typescript
// RL uses PrecedenceEngine for display values
class InsuranceRepresentation {
  async renderPolicy(policyId: string): Promise<PolicyView> {
    const sources = await this.gatherSources(policyId);
    const resolutions = await precedenceEngine.resolveAll(policyId, sources);

    return {
      policyNumber: resolutions.policy_number.final_value,
      premium: resolutions.monthly_premium.final_value,
      deductible: resolutions.deductible.final_value,
      // ... other fields

      // Include override indicators for UI
      overrides: Object.entries(resolutions)
        .filter(([_, res]) => res.metadata.is_overridden)
        .map(([field, res]) => ({
          field,
          overriddenBy: res.metadata.overridden_by,
          overriddenAt: res.metadata.overridden_at
        }))
    };
  }
}
```

#### 2. Export Services

```typescript
// Export uses PrecedenceEngine for canonical values
class AccountingExportService {
  async exportTransactions(txIds: string[]): Promise<ExportFile> {
    const batch = await this.gatherSourcesBatch(txIds);
    const resolutions = await precedenceEngine.resolveAllBatch(batch);

    const rows = [];
    for (const [txId, fields] of resolutions.entries()) {
      rows.push({
        id: txId,
        merchant: fields.merchant.final_value,
        amount: fields.amount.final_value,
        category: fields.category.final_value,
        date: fields.date.final_value
      });
    }

    return this.generateCSV(rows);
  }
}
```

#### 3. Validation Services

```typescript
// Validation uses PrecedenceEngine to verify correctness
class FieldValidationService {
  async validateDisplayedValue(
    entityId: string,
    fieldName: string,
    displayedValue: any
  ): Promise<ValidationResult> {
    const sources = await this.gatherSources(entityId, fieldName);
    const resolution = await precedenceEngine.resolveField(
      entityId,
      fieldName,
      sources
    );

    if (resolution.final_value !== displayedValue) {
      return {
        valid: false,
        error: `Displayed value "${displayedValue}" does not match resolved value "${resolution.final_value}"`
      };
    }

    return { valid: true };
  }
}
```

#### 4. Audit Services

```typescript
// Audit uses PrecedenceEngine for transparency
class AuditLogService {
  async logFieldChange(
    entityId: string,
    fieldName: string,
    oldValue: any,
    newValue: any
  ): Promise<void> {
    const sources = await this.gatherSources(entityId, fieldName);
    const resolution = await precedenceEngine.resolveField(
      entityId,
      fieldName,
      sources
    );

    const explanation = await precedenceEngine.explainResolution(
      entityId,
      fieldName,
      sources
    );

    await this.writeAuditLog({
      entity_id: entityId,
      field_name: fieldName,
      old_value: oldValue,
      new_value: newValue,
      resolution_source: resolution.source,
      explanation: explanation,
      timestamp: new Date().toISOString()
    });
  }
}
```

---

## Testing Strategy

### Unit Tests

```typescript
describe('PrecedenceEngine', () => {
  describe('resolveField', () => {
    it('should return manual override when all sources present', async () => {
      const sources: ValueSources = {
        manual_override: 'manual_value',
        rule_value: 'rule_value',
        extraction_value: 'extraction_value',
        default_value: 'default_value'
      };

      const resolution = await engine.resolveField('e1', 'f1', sources);

      expect(resolution.final_value).toBe('manual_value');
      expect(resolution.source).toBe('manual');
      expect(resolution.metadata.is_overridden).toBe(true);
    });

    it('should skip to rule when manual is undefined', async () => {
      const sources: ValueSources = {
        manual_override: undefined,
        rule_value: 'rule_value',
        extraction_value: 'extraction_value',
        default_value: 'default_value'
      };

      const resolution = await engine.resolveField('e1', 'f1', sources);

      expect(resolution.final_value).toBe('rule_value');
      expect(resolution.source).toBe('rule');
      expect(resolution.metadata.is_overridden).toBe(false);
    });

    it('should handle all sources null', async () => {
      const sources: ValueSources = {
        manual_override: null,
        rule_value: null,
        extraction_value: null,
        default_value: null
      };

      const resolution = await engine.resolveField('e1', 'f1', sources);

      expect(resolution.final_value).toBeNull();
      expect(resolution.source).toBe('manual');  // Highest priority null
    });

    it('should handle all sources undefined', async () => {
      const sources: ValueSources = {
        manual_override: undefined,
        rule_value: undefined,
        extraction_value: undefined,
        default_value: undefined
      };

      const resolution = await engine.resolveField('e1', 'f1', sources);

      expect(resolution.final_value).toBeNull();
      expect(resolution.source).toBeNull();
    });

    it('should detect override when manual differs from extraction', async () => {
      const sources: ValueSources = {
        manual_override: 'corrected_value',
        extraction_value: 'original_value'
      };

      const resolution = await engine.resolveField('e1', 'f1', sources);

      expect(resolution.metadata.is_overridden).toBe(true);
      expect(resolution.metadata.skipped_sources).toHaveLength(1);
      expect(resolution.metadata.skipped_sources[0]).toEqual({
        source: 'extraction',
        value: 'original_value'
      });
    });

    it('should mark as override even when manual equals extraction', async () => {
      const sources: ValueSources = {
        manual_override: 'same_value',
        rule_value: 'different_value',
        extraction_value: 'same_value'
      };

      const resolution = await engine.resolveField('e1', 'f1', sources);

      expect(resolution.final_value).toBe('same_value');
      expect(resolution.source).toBe('manual');
      expect(resolution.metadata.is_overridden).toBe(true);
      expect(resolution.metadata.skipped_sources).toHaveLength(1);
      expect(resolution.metadata.skipped_sources[0].source).toBe('rule');
    });
  });

  describe('resolveAll', () => {
    it('should resolve multiple fields independently', async () => {
      const fieldsWithSources = {
        field_a: { manual_override: 'manual_a' },
        field_b: { extraction_value: 'extraction_b' },
        field_c: { default_value: 'default_c' }
      };

      const resolutions = await engine.resolveAll('e1', fieldsWithSources);

      expect(resolutions.field_a.final_value).toBe('manual_a');
      expect(resolutions.field_a.source).toBe('manual');

      expect(resolutions.field_b.final_value).toBe('extraction_b');
      expect(resolutions.field_b.source).toBe('extraction');

      expect(resolutions.field_c.final_value).toBe('default_c');
      expect(resolutions.field_c.source).toBe('default');
    });
  });

  describe('resolveAllBatch', () => {
    it('should resolve multiple entities in batch', async () => {
      const batch = {
        'e1': {
          'field_a': { extraction_value: 'e1_a' }
        },
        'e2': {
          'field_a': { extraction_value: 'e2_a' }
        }
      };

      const resolutions = await engine.resolveAllBatch(batch);

      expect(resolutions.get('e1')?.field_a.final_value).toBe('e1_a');
      expect(resolutions.get('e2')?.field_a.final_value).toBe('e2_a');
    });

    it('should complete 100 entities in <100ms', async () => {
      const batch: Record<string, Record<string, ValueSources>> = {};

      for (let i = 0; i < 100; i++) {
        batch[`e${i}`] = {
          field_1: { extraction_value: `value_${i}_1` },
          field_2: { extraction_value: `value_${i}_2` },
          field_3: { extraction_value: `value_${i}_3` }
        };
      }

      const start = Date.now();
      const resolutions = await engine.resolveAllBatch(batch);
      const duration = Date.now() - start;

      expect(duration).toBeLessThan(100);
      expect(resolutions.size).toBe(100);
    });
  });

  describe('explainResolution', () => {
    it('should explain manual override', async () => {
      const sources: ValueSources = {
        manual_override: 'manual_value',
        extraction_value: 'extraction_value'
      };

      const explanation = await engine.explainResolution('e1', 'merchant', sources);

      expect(explanation).toContain('manual');
      expect(explanation).toContain('override');
      expect(explanation).toContain('extraction_value');
    });
  });
});
```

### Integration Tests

```typescript
describe('PrecedenceEngine Integration', () => {
  it('should integrate with CorrectionTracker', async () => {
    // Setup: Create manual override
    await correctionTracker.createOverride({
      entity_id: 'tx_001',
      field_name: 'merchant',
      value: 'Amazon Marketplace',
      user_id: 'user@example.com'
    });

    // Gather sources
    const sources: ValueSources = {
      manual_override: await correctionTracker.getOverride('tx_001', 'merchant'),
      extraction_value: 'AMZN MKTP'
    };

    // Resolve
    const resolution = await engine.resolveField('tx_001', 'merchant', sources);

    expect(resolution.final_value).toBe('Amazon Marketplace');
    expect(resolution.metadata.overridden_by).toBe('user@example.com');
  });

  it('should integrate with EntityStorage', async () => {
    // Setup: Store extracted entity
    await entityStorage.store({
      id: 'tx_002',
      merchant: 'STARBUCKS #1234',
      amount: 5.75
    });

    // Gather sources
    const sources: ValueSources = {
      extraction_value: await entityStorage.getField('tx_002', 'merchant')
    };

    // Resolve
    const resolution = await engine.resolveField('tx_002', 'merchant', sources);

    expect(resolution.final_value).toBe('STARBUCKS #1234');
    expect(resolution.source).toBe('extraction');
  });

  it('should handle full workflow: extract → rule → override', async () => {
    const txId = 'tx_workflow_001';

    // Step 1: Extract
    await entityStorage.store({
      id: txId,
      merchant: 'AMZN MKTP'
    });

    let sources = {
      extraction_value: await entityStorage.getField(txId, 'merchant')
    };
    let resolution = await engine.resolveField(txId, 'merchant', sources);
    expect(resolution.final_value).toBe('AMZN MKTP');
    expect(resolution.source).toBe('extraction');

    // Step 2: Apply rule
    const ruleValue = 'Amazon';
    sources = {
      rule_value: ruleValue,
      extraction_value: await entityStorage.getField(txId, 'merchant')
    };
    resolution = await engine.resolveField(txId, 'merchant', sources);
    expect(resolution.final_value).toBe('Amazon');
    expect(resolution.source).toBe('rule');

    // Step 3: Manual override
    await correctionTracker.createOverride({
      entity_id: txId,
      field_name: 'merchant',
      value: 'Amazon Marketplace',
      user_id: 'user@example.com'
    });

    sources = {
      manual_override: await correctionTracker.getOverride(txId, 'merchant'),
      rule_value: ruleValue,
      extraction_value: await entityStorage.getField(txId, 'merchant')
    };
    resolution = await engine.resolveField(txId, 'merchant', sources);
    expect(resolution.final_value).toBe('Amazon Marketplace');
    expect(resolution.source).toBe('manual');
    expect(resolution.metadata.is_overridden).toBe(true);
  });
});
```

### Performance Tests

```typescript
describe('PrecedenceEngine Performance', () => {
  it('should resolve single field in <1ms', async () => {
    const sources: ValueSources = {
      extraction_value: 'test_value'
    };

    const iterations = 1000;
    const start = Date.now();

    for (let i = 0; i < iterations; i++) {
      await engine.resolveField(`e${i}`, 'field', sources);
    }

    const duration = Date.now() - start;
    const avgDuration = duration / iterations;

    expect(avgDuration).toBeLessThan(1);
  });

  it('should resolve 100 entities in <100ms', async () => {
    const batch: Record<string, Record<string, ValueSources>> = {};

    for (let i = 0; i < 100; i++) {
      batch[`e${i}`] = {
        field_1: { extraction_value: `v${i}_1` },
        field_2: { extraction_value: `v${i}_2` },
        field_3: { extraction_value: `v${i}_3` }
      };
    }

    const start = Date.now();
    await engine.resolveAllBatch(batch);
    const duration = Date.now() - start;

    expect(duration).toBeLessThan(100);
  });

  it('should scale linearly to 1000 entities', async () => {
    const batch: Record<string, Record<string, ValueSources>> = {};

    for (let i = 0; i < 1000; i++) {
      batch[`e${i}`] = {
        field: { extraction_value: `v${i}` }
      };
    }

    const start = Date.now();
    await engine.resolveAllBatch(batch);
    const duration = Date.now() - start;

    expect(duration).toBeLessThan(500);
  });
});
```

---

## Migration Guide

### Migrating from Hard-Coded Precedence

**Before**: Precedence logic scattered across codebase

```typescript
// Old code: Manual precedence in every component
function getDisplayValue(entity: any, field: string): any {
  if (entity.overrides && entity.overrides[field]) {
    return entity.overrides[field];  // Manual check
  }

  if (entity.rules && entity.rules[field]) {
    return entity.rules[field];  // Rule check
  }

  return entity.extracted[field];  // Extraction check
}
```

**After**: Centralized PrecedenceEngine

```typescript
// New code: Use PrecedenceEngine
async function getDisplayValue(entityId: string, field: string): Promise<any> {
  const sources: ValueSources = {
    manual_override: await getOverride(entityId, field),
    rule_value: await getRuleValue(entityId, field),
    extraction_value: await getExtraction(entityId, field)
  };

  const resolution = await precedenceEngine.resolveField(entityId, field, sources);
  return resolution.final_value;
}
```

### Migration Steps

1. **Identify Precedence Logic**: Search codebase for manual precedence checks
2. **Introduce PrecedenceEngine**: Add engine to dependency injection
3. **Replace Hard-Coded Logic**: Refactor to use `resolveField()`
4. **Add Tests**: Verify behavior matches old logic
5. **Enable Audit**: Use resolution metadata for transparency
6. **Remove Old Code**: Delete manual precedence checks

### Backward Compatibility

PrecedenceEngine can wrap legacy systems:

```typescript
class LegacyPrecedenceAdapter {
  private engine = new PrecedenceEngine();

  async getFieldValue(entity: LegacyEntity, field: string): Promise<any> {
    // Adapt legacy entity to ValueSources
    const sources: ValueSources = {
      manual_override: entity.user_corrections?.[field],
      rule_value: entity.computed_values?.[field],
      extraction_value: entity.raw_data?.[field],
      default_value: this.getSchemaDefault(entity.type, field)
    };

    const resolution = await this.engine.resolveField(entity.id, field, sources);
    return resolution.final_value;
  }
}
```

---

## Related Primitives

### CorrectionTracker

- **Relationship**: Provides manual override values to PrecedenceEngine
- **Dependency**: PrecedenceEngine depends on CorrectionTracker for override data
- **Integration**: `manual_override` source comes from CorrectionTracker

```typescript
// Example integration
const override = await correctionTracker.getOverride(entityId, fieldName);

const resolution = await precedenceEngine.resolveField(entityId, fieldName, {
  manual_override: override?.value
});
```

### RuleEngine

- **Relationship**: Provides rule-generated values to PrecedenceEngine
- **Dependency**: PrecedenceEngine depends on RuleEngine for rule values
- **Integration**: `rule_value` source comes from RuleEngine

```typescript
// Example integration
const ruleResult = await ruleEngine.evaluateField(entityId, fieldName);

const resolution = await precedenceEngine.resolveField(entityId, fieldName, {
  rule_value: ruleResult?.value
});
```

### EntityStorage

- **Relationship**: Provides extracted values to PrecedenceEngine
- **Dependency**: PrecedenceEngine depends on EntityStorage for extraction data
- **Integration**: `extraction_value` source comes from EntityStorage

```typescript
// Example integration
const extractedValue = await entityStorage.getField(entityId, fieldName);

const resolution = await precedenceEngine.resolveField(entityId, fieldName, {
  extraction_value: extractedValue
});
```

### Representation Layer

- **Relationship**: Consumes PrecedenceEngine for display values
- **Dependency**: Representation Layer depends on PrecedenceEngine
- **Integration**: RL calls PrecedenceEngine to resolve all fields before display

```typescript
// Example integration
class FinanceRepresentation {
  async renderTransaction(txId: string): Promise<TransactionView> {
    const sources = await this.gatherAllSources(txId);
    const resolutions = await precedenceEngine.resolveAll(txId, sources);

    return {
      merchant: resolutions.merchant.final_value,
      amount: resolutions.amount.final_value,
      // ... other fields
    };
  }
}
```

---

## Appendix

### A. Complete Implementation Example

```typescript
import { v4 as uuidv4 } from 'uuid';

/**
 * Production-ready PrecedenceEngine implementation.
 */
export class PrecedenceEngineImpl implements PrecedenceEngine {
  private readonly precedenceRules: PrecedenceRule[] = [
    {
      priority: 1,
      source: 'manual',
      description: 'Manual override set by user',
      conditions: ['value !== undefined', 'value !== null']
    },
    {
      priority: 2,
      source: 'rule',
      description: 'Value generated by automated rule',
      conditions: ['value !== undefined', 'value !== null']
    },
    {
      priority: 3,
      source: 'extraction',
      description: 'Value extracted from source document',
      conditions: ['value !== undefined', 'value !== null']
    },
    {
      priority: 4,
      source: 'default',
      description: 'Default/fallback value from schema',
      conditions: ['value !== undefined', 'value !== null']
    }
  ];

  async resolveField(
    entityId: string,
    fieldName: string,
    sources: ValueSources
  ): Promise<PrecedenceResolution> {
    // Validate inputs
    this.validateInputs(entityId, fieldName, sources);

    // Apply precedence rules
    const { finalValue, finalSource } = this.applyPrecedence(sources);

    // Detect override
    const isOverridden = this.detectOverride(sources, finalSource);

    // Build skipped sources list
    const skippedSources = this.buildSkippedSources(sources, finalSource);

    return {
      field_name: fieldName,
      final_value: finalValue,
      source: finalSource,
      sources: sources,
      metadata: {
        is_overridden: isOverridden,
        skipped_sources: skippedSources
      }
    };
  }

  async resolveAll(
    entityId: string,
    fieldsWithSources: Record<string, ValueSources>
  ): Promise<Record<string, PrecedenceResolution>> {
    const resolutions: Record<string, PrecedenceResolution> = {};

    for (const [fieldName, sources] of Object.entries(fieldsWithSources)) {
      resolutions[fieldName] = await this.resolveField(
        entityId,
        fieldName,
        sources
      );
    }

    return resolutions;
  }

  async resolveAllBatch(
    batch: Record<string, Record<string, ValueSources>>
  ): Promise<Map<string, Record<string, PrecedenceResolution>>> {
    const results = new Map<string, Record<string, PrecedenceResolution>>();

    // Process in parallel for performance
    const promises = Object.entries(batch).map(async ([entityId, fields]) => {
      const resolutions = await this.resolveAll(entityId, fields);
      return [entityId, resolutions] as const;
    });

    const resolved = await Promise.all(promises);

    for (const [entityId, resolutions] of resolved) {
      results.set(entityId, resolutions);
    }

    return results;
  }

  getPrecedenceRules(): PrecedenceRule[] {
    return [...this.precedenceRules];
  }

  async explainResolution(
    entityId: string,
    fieldName: string,
    sources: ValueSources
  ): Promise<string> {
    const resolution = await this.resolveField(entityId, fieldName, sources);

    let explanation = `Field '${fieldName}' resolved to '${resolution.final_value}'`;

    if (resolution.source === null) {
      explanation += ' because no value sources are available.';
      return explanation;
    }

    explanation += ` because:\n`;

    switch (resolution.source) {
      case 'manual':
        explanation += `  1. A manual override was set`;
        if (resolution.metadata.overridden_by) {
          explanation += ` by ${resolution.metadata.overridden_by}`;
        }
        if (resolution.metadata.overridden_at) {
          explanation += ` on ${resolution.metadata.overridden_at}`;
        }
        break;

      case 'rule':
        explanation += `  1. An automated rule generated this value`;
        break;

      case 'extraction':
        explanation += `  1. This value was extracted from the source document`;
        break;

      case 'default':
        explanation += `  1. This is the default value (no other sources available)`;
        break;
    }

    if (resolution.metadata.skipped_sources &&
        resolution.metadata.skipped_sources.length > 0) {
      explanation += `\n\nSkipped lower-priority sources:\n`;
      for (const skipped of resolution.metadata.skipped_sources) {
        explanation += `  - ${skipped.source}: '${skipped.value}'\n`;
      }
    }

    return explanation;
  }

  // Private helper methods

  private validateInputs(
    entityId: string,
    fieldName: string,
    sources: ValueSources
  ): void {
    if (!entityId) {
      throw new Error('entityId is required');
    }

    if (!fieldName) {
      throw new Error('fieldName is required');
    }

    if (!sources || typeof sources !== 'object') {
      throw new Error('sources must be an object');
    }
  }

  private applyPrecedence(sources: ValueSources): {
    finalValue: any;
    finalSource: 'manual' | 'rule' | 'extraction' | 'default' | null;
  } {
    // Check manual override
    if (sources.manual_override !== undefined && sources.manual_override !== null) {
      return { finalValue: sources.manual_override, finalSource: 'manual' };
    }

    // Check rule value
    if (sources.rule_value !== undefined && sources.rule_value !== null) {
      return { finalValue: sources.rule_value, finalSource: 'rule' };
    }

    // Check extraction value
    if (sources.extraction_value !== undefined && sources.extraction_value !== null) {
      return { finalValue: sources.extraction_value, finalSource: 'extraction' };
    }

    // Check default value
    if (sources.default_value !== undefined && sources.default_value !== null) {
      return { finalValue: sources.default_value, finalSource: 'default' };
    }

    // Handle explicit nulls (check manual first, then others)
    if (sources.manual_override === null) {
      return { finalValue: null, finalSource: 'manual' };
    }

    if (sources.rule_value === null) {
      return { finalValue: null, finalSource: 'rule' };
    }

    if (sources.extraction_value === null) {
      return { finalValue: null, finalSource: 'extraction' };
    }

    if (sources.default_value === null) {
      return { finalValue: null, finalSource: 'default' };
    }

    // No sources available
    return { finalValue: null, finalSource: null };
  }

  private detectOverride(
    sources: ValueSources,
    finalSource: string | null
  ): boolean {
    if (finalSource !== 'manual') {
      return false;
    }

    const manualValue = sources.manual_override;
    const lowerPrioritySources = [
      sources.rule_value,
      sources.extraction_value,
      sources.default_value
    ];

    for (const lowerValue of lowerPrioritySources) {
      if (lowerValue !== undefined &&
          lowerValue !== null &&
          !this.deepEqual(lowerValue, manualValue)) {
        return true;
      }
    }

    return false;
  }

  private buildSkippedSources(
    sources: ValueSources,
    finalSource: string | null
  ): Array<{ source: string; value: any }> {
    const skipped: Array<{ source: string; value: any }> = [];

    const sourceOrder: Array<{ key: keyof ValueSources; name: string }> = [
      { key: 'manual_override', name: 'manual' },
      { key: 'rule_value', name: 'rule' },
      { key: 'extraction_value', name: 'extraction' },
      { key: 'default_value', name: 'default' }
    ];

    let foundFinal = false;

    for (const { key, name } of sourceOrder) {
      if (name === finalSource) {
        foundFinal = true;
        continue;
      }

      if (foundFinal &&
          sources[key] !== undefined &&
          sources[key] !== null) {
        skipped.push({
          source: name,
          value: sources[key]
        });
      }
    }

    return skipped;
  }

  private deepEqual(a: any, b: any): boolean {
    return JSON.stringify(a) === JSON.stringify(b);
  }
}
```

### B. Glossary

- **Precedence**: Priority order for selecting a value from multiple sources
- **Manual Override**: Value explicitly set by a user, highest priority
- **Rule Value**: Value generated by automated business logic
- **Extraction Value**: Value parsed from source document (OCR, API, file)
- **Default Value**: Fallback value defined in schema
- **Resolution**: Process of determining final value using precedence
- **Override Detection**: Identifying when manual value differs from other sources
- **Stateless**: No internal state retained between calls
- **Batch Resolution**: Resolving multiple entities in a single operation

### C. References

- **Corrections Flow Documentation**: `docs/workflows/corrections-flow.md`
- **CorrectionTracker Primitive**: `docs/primitives/ol/CorrectionTracker.md`
- **EntityStorage Primitive**: `docs/primitives/ol/EntityStorage.md`
- **RuleEngine Primitive**: `docs/primitives/ol/RuleEngine.md`
- **Representation Layer Architecture**: `docs/architecture/representation-layer.md`

### D. Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-10-24 | Initial specification |

### E. Frequently Asked Questions

**Q: Can precedence order be customized per field?**
A: No. Precedence order is universal and static. Domain-specific logic should be implemented in the RuleEngine, not by changing precedence.

**Q: What happens if manual override equals extraction value?**
A: It's still treated as an override (user confirmed the value). `is_overridden` will be `true`, but `skipped_sources` will only list different values.

**Q: How does PrecedenceEngine handle type mismatches?**
A: It doesn't. PrecedenceEngine returns values as-is without type coercion. Type validation is the responsibility of the caller.

**Q: Can I cache PrecedenceEngine resolutions?**
A: The engine itself is stateless, but you can implement caching at the caller level. Be careful to invalidate cache when override/rule/extraction values change.

**Q: Does PrecedenceEngine query the database?**
A: No. PrecedenceEngine operates purely in-memory on pre-fetched values. The caller is responsible for gathering sources from upstream systems.

**Q: What's the difference between `null` and `undefined` in sources?**
A: `undefined` means "source not available" (skip to next priority). `null` means "source explicitly says no value" (participates in precedence).

**Q: How do I optimize batch operations?**
A: Fetch all sources upfront in batched queries (1-2 queries), then pass to `resolveAllBatch()`. Avoid fetching sources inside loops.

---

**End of Specification**

*This document is part of the Personal OS v1 Objective Layer primitive library.*
