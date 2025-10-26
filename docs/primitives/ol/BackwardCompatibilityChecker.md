# BackwardCompatibilityChecker OL Primitive

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
8. [Breaking Change Detection](#breaking-change-detection)
9. [Additive Change Detection](#additive-change-detection)
10. [Compatibility Rules](#compatibility-rules)
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

The **BackwardCompatibilityChecker** detects breaking changes between schema versions by analyzing structural and semantic differences. It provides detailed compatibility reports with categorized changes (breaking, additive, non-functional) to guide version bumping and migration planning.

### Key Capabilities

- **Breaking Change Detection**: Identifies changes that break backward compatibility
- **Additive Change Detection**: Identifies backward-compatible additions
- **Detailed Change Reports**: Categorizes all changes with descriptions
- **Constraint Analysis**: Detects constraint tightening/relaxing
- **Type Compatibility**: Validates type changes and coercions
- **Nested Field Analysis**: Analyzes changes in nested object structures
- **Enum Value Tracking**: Detects added/removed enum values

### Design Philosophy

The BackwardCompatibilityChecker follows four core principles:

1. **Comprehensive Detection**: Catches all breaking changes, no false negatives
2. **Clear Categorization**: Separates breaking, additive, and non-functional changes
3. **Actionable Reports**: Provides specific guidance on impact and mitigation
4. **Conservative Analysis**: When in doubt, classify as breaking (safety first)

---

## Purpose & Scope

### Problem Statement

In schema evolution, detecting breaking changes is critical but challenging:

- **Manual Review Error-Prone**: Developers miss subtle breaking changes
- **No Automated Detection**: Schema changes deployed without compatibility checks
- **Unclear Impact**: Developers unsure if change breaks backward compatibility
- **Downtime Risk**: Breaking changes break production consumers
- **No Change Categorization**: All changes treated equally

Traditional approaches have significant limitations:

**Approach 1: Manual Code Review**
- ❌ Time-consuming and error-prone
- ❌ Inconsistent across reviewers
- ❌ Misses subtle breaking changes

**Approach 2: Runtime Validation**
- ❌ Detects issues AFTER deployment
- ❌ Requires production traffic to detect
- ❌ Can't prevent breaking changes

**Approach 3: BackwardCompatibilityChecker**
- ✅ Automated breaking change detection
- ✅ Comprehensive structural analysis
- ✅ Categorized change reports
- ✅ Detects issues BEFORE deployment
- ✅ Actionable recommendations

### Solution

The BackwardCompatibilityChecker implements **automated compatibility analysis** with:

1. **Schema Diffing**: Deep comparison of schema structures
2. **Change Categorization**: Breaking, additive, non-functional
3. **Constraint Analysis**: Detect tightened/relaxed constraints
4. **Type Checking**: Validate type changes and coercions
5. **Detailed Reports**: Generate actionable compatibility reports

---

## Multi-Domain Applicability

The BackwardCompatibilityChecker is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Validate bank transaction schema changes before deployment.

**Breaking Changes Detected**:
- Remove `merchant` field → Breaks consumers expecting field
- Change `amount` type from number to string → Type incompatibility
- Make `category` field required → Breaks old data without category

**Additive Changes Detected**:
- Add optional `merchant_category_code` field → Backward compatible
- Relax `amount` minimum from 0.01 to 0 → Backward compatible

**Example**:
```typescript
const checker = new BackwardCompatibilityChecker();

const oldSchema = {
  type: "object",
  properties: {
    transaction_id: { type: "string" },
    amount: { type: "number", minimum: 0.01 },
    merchant: { type: "string" }
  },
  required: ["transaction_id", "amount"]
};

const newSchema = {
  type: "object",
  properties: {
    transaction_id: { type: "string" },
    amount: { type: "number", minimum: 0 }, // Relaxed constraint
    merchant: { type: "string" },
    merchant_category_code: { type: "string" } // Added optional field
  },
  required: ["transaction_id", "amount"]
};

const report = await checker.check(oldSchema, newSchema);

console.log(report);
// {
//   compatible: true,
//   breaking_changes: [],
//   additive_changes: [
//     {
//       type: "FIELD_ADDED",
//       field: "merchant_category_code",
//       severity: "ADDITIVE",
//       description: "Optional field 'merchant_category_code' added"
//     },
//     {
//       type: "CONSTRAINT_RELAXED",
//       field: "amount",
//       constraint: "minimum",
//       old_value: 0.01,
//       new_value: 0,
//       severity: "ADDITIVE",
//       description: "Minimum constraint relaxed from 0.01 to 0"
//     }
//   ],
//   non_functional_changes: [],
//   recommendation: "MINOR version bump (backward compatible additions)"
// }
```

**Compliance**: SOX requires compatibility validation before schema changes.

---

### 2. Healthcare

**Use Case**: Validate FHIR patient schema changes for backward compatibility.

**Breaking Changes Detected**:
- Remove `deceasedBoolean` field → Breaks consumers using field
- Change `gender` from optional to required → Breaks old data
- Change `birthDate` type from string to object → Type incompatibility

**Additive Changes Detected**:
- Add optional `race` field → Backward compatible
- Add new enum value to `gender` → Backward compatible

**Example**:
```typescript
// Detect breaking change: remove field
const oldSchema = {
  type: "object",
  properties: {
    id: { type: "string" },
    name: { type: "string" },
    deceasedBoolean: { type: "boolean" } // Will be removed
  }
};

const newSchema = {
  type: "object",
  properties: {
    id: { type: "string" },
    name: { type: "string" }
    // deceasedBoolean removed
  }
};

const report = await checker.check(oldSchema, newSchema);

console.log(report.breaking_changes);
// [
//   {
//     type: "FIELD_REMOVED",
//     field: "deceasedBoolean",
//     severity: "BREAKING",
//     description: "Field 'deceasedBoolean' removed",
//     impact: "Consumers expecting this field will fail",
//     recommendation: "MAJOR version bump required"
//   }
// ]
```

**Compliance**: HIPAA requires validation of PHI schema changes.

---

### 3. Legal

**Use Case**: Validate court filing schema changes for e-discovery systems.

**Breaking Changes Detected**:
- Change `case_number` regex pattern (more restrictive) → Breaks old data
- Remove `exhibits` field → Breaks consumers
- Change `filing_date` from string to ISO format → Format incompatibility

**Additive Changes Detected**:
- Add optional `jurisdiction` field → Backward compatible
- Add new `motion` value to `document_type` enum → Backward compatible

**Example**:
```typescript
// Detect breaking change: tighten constraint
const oldSchema = {
  type: "object",
  properties: {
    case_number: {
      type: "string",
      pattern: "^[0-9]+-CV-[0-9]+$" // Less restrictive
    }
  }
};

const newSchema = {
  type: "object",
  properties: {
    case_number: {
      type: "string",
      pattern: "^[0-9]{2}-CV-[0-9]{5}$" // More restrictive (exact length)
    }
  }
};

const report = await checker.check(oldSchema, newSchema);

console.log(report.breaking_changes);
// [
//   {
//     type: "CONSTRAINT_TIGHTENED",
//     field: "case_number",
//     constraint: "pattern",
//     old_value: "^[0-9]+-CV-[0-9]+$",
//     new_value: "^[0-9]{2}-CV-[0-9]{5}$",
//     severity: "BREAKING",
//     description: "Pattern constraint tightened",
//     impact: "Old data matching old pattern may not match new pattern"
//   }
// ]
```

---

### 4. Research (RSRCH - Utilitario)

**Use Case**: Validate founder fact schema changes for RSRCH utilitario research system.

**Breaking Changes Detected**:
- Change `subject_entity` from string to object → Structure incompatibility
- Remove `source_url` field → Breaks consumers
- Change `sources` from array to single object → Structure incompatibility

**Additive Changes Detected**:
- Add optional `confidence` field for fact credibility → Backward compatible
- Add `investment_amount` field for investment facts → Backward compatible

**Example**:
```typescript
// Detect breaking change: subject_entity type change
const oldSchema = {
  type: "object",
  properties: {
    claim: { type: "string" },
    subject_entity: { type: "string" } // String type (e.g., "Sam Altman")
  }
};

const newSchema = {
  type: "object",
  properties: {
    claim: { type: "string" },
    subject_entity: { type: "object" } // Changed to object (entity_id, entity_name, entity_type)
  }
};

const report = await checker.check(oldSchema, newSchema);

console.log(report.breaking_changes);
// [
//   {
//     type: "TYPE_CHANGED",
//     field: "year",
//     old_type: "string",
//     new_type: "integer",
//     severity: "BREAKING",
//     description: "Field type changed from string to integer",
//     impact: "Old data with string values will fail validation"
//   }
// ]
```

---

### 5. E-commerce

**Use Case**: Validate product catalog schema changes.

**Breaking Changes Detected**:
- Make `brand` field required → Breaks old data without brand
- Remove `legacy_sku` field → Breaks consumers
- Change `price` precision (increase minimum) → Breaks low-priced items

**Additive Changes Detected**:
- Add optional `variants` array → Backward compatible
- Add `on_sale` boolean field → Backward compatible

**Example**:
```typescript
// Detect breaking change: add required field
const oldSchema = {
  type: "object",
  properties: {
    sku: { type: "string" },
    name: { type: "string" },
    price: { type: "number" }
  },
  required: ["sku", "name", "price"]
};

const newSchema = {
  type: "object",
  properties: {
    sku: { type: "string" },
    name: { type: "string" },
    price: { type: "number" },
    brand: { type: "string" } // New field
  },
  required: ["sku", "name", "price", "brand"] // Added to required
};

const report = await checker.check(oldSchema, newSchema);

console.log(report.breaking_changes);
// [
//   {
//     type: "FIELD_REQUIRED_ADDED",
//     field: "brand",
//     severity: "BREAKING",
//     description: "Field 'brand' is now required",
//     impact: "Old data without 'brand' field will fail validation"
//   }
// ]
```

---

### 6. SaaS

**Use Case**: Validate API schema changes for subscription platform.

**Breaking Changes Detected**:
- Remove `trial_days` field → Breaks API consumers
- Change `plan` from string to object → Structure incompatibility
- Make `payment_method` required → Breaks trial accounts

**Additive Changes Detected**:
- Add optional `features` array → Backward compatible
- Add `cancelled_at` timestamp → Backward compatible

**Example**:
```typescript
// Detect breaking change: structure change
const oldSchema = {
  type: "object",
  properties: {
    subscription_id: { type: "string" },
    plan: { type: "string" } // Simple string
  }
};

const newSchema = {
  type: "object",
  properties: {
    subscription_id: { type: "string" },
    plan: { // Changed to object
      type: "object",
      properties: {
        id: { type: "string" },
        name: { type: "string" }
      }
    }
  }
};

const report = await checker.check(oldSchema, newSchema);

console.log(report.breaking_changes);
// [
//   {
//     type: "TYPE_CHANGED",
//     field: "plan",
//     old_type: "string",
//     new_type: "object",
//     severity: "BREAKING",
//     description: "Field type changed from string to object",
//     impact: "Consumers expecting string will fail"
//   }
// ]
```

---

### 7. Insurance

**Use Case**: Validate insurance policy schema changes.

**Breaking Changes Detected**:
- Remove `legacy_policy_number` field → Breaks legacy integrations
- Change `premium` calculation (add required fields) → Breaks old policies
- Change `deductible` minimum (increase) → Breaks low-deductible policies

**Additive Changes Detected**:
- Add optional `co_pay` field → Backward compatible
- Add `renewal_date` field → Backward compatible

**Example**:
```typescript
// Detect breaking change: increase minimum
const oldSchema = {
  type: "object",
  properties: {
    deductible: {
      type: "number",
      minimum: 0
    }
  }
};

const newSchema = {
  type: "object",
  properties: {
    deductible: {
      type: "number",
      minimum: 500 // Increased from 0
    }
  }
};

const report = await checker.check(oldSchema, newSchema);

console.log(report.breaking_changes);
// [
//   {
//     type: "CONSTRAINT_TIGHTENED",
//     field: "deductible",
//     constraint: "minimum",
//     old_value: 0,
//     new_value: 500,
//     severity: "BREAKING",
//     description: "Minimum constraint increased from 0 to 500",
//     impact: "Policies with deductible < 500 will fail validation"
//   }
// ]
```

---

### Cross-Domain Benefits

**Automated Validation**: All domains benefit from:
- Breaking change detection before deployment
- Categorized change reports
- Actionable recommendations
- Prevents production incidents

**Regulatory Compliance**: Supports:
- SOX (Finance): Schema change validation
- HIPAA (Healthcare): PHI schema integrity
- GDPR (Privacy): Data structure compliance
- Legal Discovery: Schema evolution tracking

---

## Core Concepts

### Backward Compatibility

A schema change is **backward compatible** if:
1. Old data remains valid under new schema
2. Consumers using old schema can read new data
3. No breaking changes introduced

### Breaking Changes

Changes that violate backward compatibility:

**Field Operations**:
- Remove field
- Rename field (without alias)
- Change field type (non-coercible)

**Requirement Changes**:
- Add required field (without default)
- Remove optional field that was relied upon

**Constraint Changes**:
- Tighten constraint (increase minimum, decrease maximum)
- Change pattern to more restrictive
- Remove enum value

### Additive Changes

Changes that maintain backward compatibility:

**Field Operations**:
- Add optional field
- Add field with default value

**Constraint Changes**:
- Relax constraint (decrease minimum, increase maximum)
- Add enum value

**Type Changes**:
- Widen type (integer → number)

### Non-Functional Changes

Changes with no compatibility impact:
- Update description
- Update examples
- Fix typos
- Reorder fields (JSON Schema)

---

## Interface Definition

### TypeScript Interface

```typescript
interface BackwardCompatibilityChecker {
  /**
   * Check backward compatibility between two schemas
   *
   * @param oldSchema - Previous schema version
   * @param newSchema - New schema version
   * @returns Compatibility report
   */
  check(oldSchema: object, newSchema: object): Promise<CompatibilityReport>;

  /**
   * Detect breaking changes between schemas
   *
   * @param oldSchema - Previous schema version
   * @param newSchema - New schema version
   * @returns Array of breaking changes
   */
  detectBreakingChanges(
    oldSchema: object,
    newSchema: object
  ): Promise<BreakingChange[]>;

  /**
   * Detect additive changes between schemas
   *
   * @param oldSchema - Previous schema version
   * @param newSchema - New schema version
   * @returns Array of additive changes
   */
  detectAdditiveChanges(
    oldSchema: object,
    newSchema: object
  ): Promise<AdditiveChange[]>;

  /**
   * Generate detailed compatibility report
   *
   * @param oldSchema - Previous schema version
   * @param newSchema - New schema version
   * @returns Detailed report with recommendations
   */
  generateReport(
    oldSchema: object,
    newSchema: object
  ): Promise<DetailedCompatibilityReport>;

  /**
   * Check if specific field change is breaking
   *
   * @param oldField - Previous field definition
   * @param newField - New field definition
   * @param fieldName - Field name
   * @returns true if breaking change
   */
  isFieldChangeBreaking(
    oldField: object,
    newField: object,
    fieldName: string
  ): boolean;

  /**
   * Detect constraint changes
   *
   * @param oldSchema - Previous schema version
   * @param newSchema - New schema version
   * @returns Array of constraint changes
   */
  detectConstraintChanges(
    oldSchema: object,
    newSchema: object
  ): Promise<ConstraintChange[]>;

  /**
   * Validate schema evolution path
   *
   * @param schemas - Array of schema versions in order
   * @returns Validation result
   */
  validateEvolutionPath(schemas: object[]): Promise<EvolutionValidation>;
}
```

---

## Data Model

### CompatibilityReport Type

```typescript
interface CompatibilityReport {
  compatible: boolean;                      // Overall compatibility
  breaking_changes: BreakingChange[];       // Breaking changes
  additive_changes: AdditiveChange[];       // Additive changes
  non_functional_changes: Change[];         // Non-functional changes
  recommendation: string;                   // Version bump recommendation
  risk_level: RiskLevel;                    // LOW | MEDIUM | HIGH
}
```

### BreakingChange Type

```typescript
interface BreakingChange {
  type: BreakingChangeType;                 // Type of breaking change
  field?: string;                           // Field name (if applicable)
  severity: "BREAKING";                     // Always BREAKING
  description: string;                      // Human-readable description
  impact: string;                           // Impact on consumers
  recommendation: string;                   // Recommended action
  old_value?: any;                          // Previous value
  new_value?: any;                          // New value
}

type BreakingChangeType =
  | "FIELD_REMOVED"
  | "FIELD_REQUIRED_ADDED"
  | "TYPE_CHANGED"
  | "CONSTRAINT_TIGHTENED"
  | "ENUM_VALUE_REMOVED"
  | "FIELD_RENAMED";
```

### AdditiveChange Type

```typescript
interface AdditiveChange {
  type: AdditiveChangeType;                 // Type of additive change
  field?: string;                           // Field name (if applicable)
  severity: "ADDITIVE";                     // Always ADDITIVE
  description: string;                      // Human-readable description
  old_value?: any;                          // Previous value
  new_value?: any;                          // New value
}

type AdditiveChangeType =
  | "FIELD_ADDED"
  | "CONSTRAINT_RELAXED"
  | "ENUM_VALUE_ADDED"
  | "OPTIONAL_FIELD_ADDED";
```

### ConstraintChange Type

```typescript
interface ConstraintChange {
  field: string;                            // Field name
  constraint: string;                       // Constraint name (minimum, maximum, pattern, etc.)
  old_value: any;                           // Old constraint value
  new_value: any;                           // New constraint value
  breaking: boolean;                        // Whether change is breaking
  description: string;                      // Human-readable description
}
```

### DetailedCompatibilityReport Type

```typescript
interface DetailedCompatibilityReport extends CompatibilityReport {
  schema_diff: SchemaDiff;                  // Detailed schema diff
  affected_fields: string[];                // List of affected fields
  migration_required: boolean;              // Whether data migration needed
  estimated_impact: ImpactEstimate;         // Estimated impact
}
```

---

## Core Functionality

### 1. check()

Check backward compatibility between two schemas.

#### Signature

```typescript
check(oldSchema: object, newSchema: object): Promise<CompatibilityReport>
```

#### Behavior

1. Compare schema structures
2. Detect breaking changes
3. Detect additive changes
4. Detect non-functional changes
5. Generate compatibility report with recommendations

#### Example

```typescript
const checker = new BackwardCompatibilityChecker();

const oldSchema = {
  type: "object",
  properties: {
    id: { type: "string" },
    name: { type: "string" }
  },
  required: ["id"]
};

const newSchema = {
  type: "object",
  properties: {
    id: { type: "string" },
    name: { type: "string" },
    email: { type: "string" }
  },
  required: ["id", "email"] // email now required
};

const report = await checker.check(oldSchema, newSchema);

console.log(report);
// {
//   compatible: false,
//   breaking_changes: [
//     {
//       type: "FIELD_REQUIRED_ADDED",
//       field: "email",
//       severity: "BREAKING",
//       description: "Field 'email' is now required",
//       impact: "Old data without 'email' will fail validation",
//       recommendation: "MAJOR version bump required"
//     }
//   ],
//   additive_changes: [],
//   non_functional_changes: [],
//   recommendation: "MAJOR version bump (breaking changes detected)",
//   risk_level: "HIGH"
// }
```

---

### 2. detectBreakingChanges()

Detect breaking changes between schemas.

#### Signature

```typescript
detectBreakingChanges(
  oldSchema: object,
  newSchema: object
): Promise<BreakingChange[]>
```

#### Behavior

1. Compare field definitions
2. Identify removed fields
3. Identify type changes
4. Identify tightened constraints
5. Return array of breaking changes

#### Example

```typescript
const breakingChanges = await checker.detectBreakingChanges(oldSchema, newSchema);

console.log(breakingChanges);
// [
//   {
//     type: "FIELD_REMOVED",
//     field: "legacy_id",
//     severity: "BREAKING",
//     description: "Field 'legacy_id' removed",
//     impact: "Consumers expecting this field will fail"
//   },
//   {
//     type: "TYPE_CHANGED",
//     field: "age",
//     old_type: "string",
//     new_type: "integer",
//     severity: "BREAKING",
//     description: "Field type changed from string to integer"
//   }
// ]
```

---

### 3. detectAdditiveChanges()

Detect additive (non-breaking) changes between schemas.

#### Signature

```typescript
detectAdditiveChanges(
  oldSchema: object,
  newSchema: object
): Promise<AdditiveChange[]>
```

#### Behavior

1. Identify added optional fields
2. Identify relaxed constraints
3. Identify added enum values
4. Return array of additive changes

#### Example

```typescript
const additiveChanges = await checker.detectAdditiveChanges(oldSchema, newSchema);

console.log(additiveChanges);
// [
//   {
//     type: "FIELD_ADDED",
//     field: "middle_name",
//     severity: "ADDITIVE",
//     description: "Optional field 'middle_name' added"
//   },
//   {
//     type: "CONSTRAINT_RELAXED",
//     field: "age",
//     constraint: "minimum",
//     old_value: 18,
//     new_value: 0,
//     severity: "ADDITIVE",
//     description: "Minimum constraint relaxed from 18 to 0"
//   }
// ]
```

---

### 4. generateReport()

Generate detailed compatibility report with recommendations.

#### Signature

```typescript
generateReport(
  oldSchema: object,
  newSchema: object
): Promise<DetailedCompatibilityReport>
```

#### Behavior

1. Run full compatibility check
2. Generate schema diff
3. Estimate migration impact
4. Provide detailed recommendations

#### Example

```typescript
const report = await checker.generateReport(oldSchema, newSchema);

console.log(report);
// {
//   compatible: true,
//   breaking_changes: [],
//   additive_changes: [
//     { type: "FIELD_ADDED", field: "email", ... }
//   ],
//   non_functional_changes: [],
//   recommendation: "MINOR version bump",
//   risk_level: "LOW",
//   schema_diff: {
//     added_fields: ["email"],
//     removed_fields: [],
//     modified_fields: []
//   },
//   affected_fields: ["email"],
//   migration_required: false,
//   estimated_impact: {
//     consumers_affected: 0,
//     data_migration_needed: false,
//     rollback_difficulty: "EASY"
//   }
// }
```

---

### 5. isFieldChangeBreaking()

Check if specific field change is breaking.

#### Signature

```typescript
isFieldChangeBreaking(
  oldField: object,
  newField: object,
  fieldName: string
): boolean
```

#### Behavior

1. Compare field types
2. Compare field constraints
3. Check required status
4. Return true if breaking

#### Example

```typescript
const oldField = { type: "string" };
const newField = { type: "integer" };

const isBreaking = checker.isFieldChangeBreaking(oldField, newField, "age");

console.log(isBreaking); // true (type changed)
```

---

### 6. detectConstraintChanges()

Detect constraint changes (tightened or relaxed).

#### Signature

```typescript
detectConstraintChanges(
  oldSchema: object,
  newSchema: object
): Promise<ConstraintChange[]>
```

#### Behavior

1. Compare numeric constraints (minimum, maximum)
2. Compare string constraints (minLength, maxLength, pattern)
3. Compare array constraints (minItems, maxItems)
4. Categorize as tightened (breaking) or relaxed (additive)

#### Example

```typescript
const constraintChanges = await checker.detectConstraintChanges(oldSchema, newSchema);

console.log(constraintChanges);
// [
//   {
//     field: "age",
//     constraint: "minimum",
//     old_value: 0,
//     new_value: 18,
//     breaking: true, // Tightened
//     description: "Minimum increased from 0 to 18"
//   },
//   {
//     field: "name",
//     constraint: "maxLength",
//     old_value: 50,
//     new_value: 100,
//     breaking: false, // Relaxed
//     description: "MaxLength increased from 50 to 100"
//   }
// ]
```

---

### 7. validateEvolutionPath()

Validate schema evolution path across multiple versions.

#### Signature

```typescript
validateEvolutionPath(schemas: object[]): Promise<EvolutionValidation>
```

#### Behavior

1. Check compatibility between each consecutive pair
2. Identify any breaking changes in evolution path
3. Return validation result

#### Example

```typescript
const schemas = [
  schemaV1_0_0,
  schemaV1_1_0,
  schemaV2_0_0,
  schemaV2_1_0
];

const validation = await checker.validateEvolutionPath(schemas);

console.log(validation);
// {
//   valid: true,
//   breaking_steps: [
//     {
//       from_version: "1.1.0",
//       to_version: "2.0.0",
//       breaking_changes: [
//         { type: "FIELD_REMOVED", field: "legacy_id" }
//       ]
//     }
//   ],
//   total_changes: 15,
//   recommendation: "Evolution path valid (1 breaking change at v2.0.0)"
// }
```

---

## Breaking Change Detection

### Breaking Change Rules

#### Rule 1: Field Removed

```typescript
// BREAKING: Field removed
oldSchema: {
  properties: {
    id: { type: "string" },
    legacy_id: { type: "string" } // This field
  }
}

newSchema: {
  properties: {
    id: { type: "string" }
    // legacy_id removed
  }
}

// Detection
if (oldField exists && newField does not exist) {
  breakingChange("FIELD_REMOVED", field);
}
```

#### Rule 2: Field Type Changed

```typescript
// BREAKING: Type changed (non-coercible)
oldSchema: {
  properties: {
    age: { type: "string" }
  }
}

newSchema: {
  properties: {
    age: { type: "integer" } // Type changed
  }
}

// Detection
if (oldField.type !== newField.type && !isCoercible(oldType, newType)) {
  breakingChange("TYPE_CHANGED", field);
}
```

#### Rule 3: Required Field Added

```typescript
// BREAKING: Field added to required array
oldSchema: {
  required: ["id"]
}

newSchema: {
  required: ["id", "email"] // email added
}

// Detection
if (newRequired.includes(field) && !oldRequired.includes(field)) {
  breakingChange("FIELD_REQUIRED_ADDED", field);
}
```

#### Rule 4: Constraint Tightened

```typescript
// BREAKING: Minimum increased
oldSchema: {
  properties: {
    age: { type: "integer", minimum: 0 }
  }
}

newSchema: {
  properties: {
    age: { type: "integer", minimum: 18 } // Increased
  }
}

// Detection
if (newMinimum > oldMinimum) {
  breakingChange("CONSTRAINT_TIGHTENED", field);
}
```

#### Rule 5: Enum Value Removed

```typescript
// BREAKING: Enum value removed
oldSchema: {
  properties: {
    status: { type: "string", enum: ["active", "inactive", "pending"] }
  }
}

newSchema: {
  properties: {
    status: { type: "string", enum: ["active", "inactive"] } // "pending" removed
  }
}

// Detection
if (oldEnumValues.includes(value) && !newEnumValues.includes(value)) {
  breakingChange("ENUM_VALUE_REMOVED", field);
}
```

---

## Additive Change Detection

### Additive Change Rules

#### Rule 1: Optional Field Added

```typescript
// ADDITIVE: Optional field added
oldSchema: {
  properties: {
    id: { type: "string" }
  }
}

newSchema: {
  properties: {
    id: { type: "string" },
    email: { type: "string" } // New optional field
  }
}

// Detection
if (newField exists && oldField does not exist && !isRequired(field)) {
  additiveChange("FIELD_ADDED", field);
}
```

#### Rule 2: Constraint Relaxed

```typescript
// ADDITIVE: Minimum decreased
oldSchema: {
  properties: {
    age: { type: "integer", minimum: 18 }
  }
}

newSchema: {
  properties: {
    age: { type: "integer", minimum: 0 } // Decreased
  }
}

// Detection
if (newMinimum < oldMinimum) {
  additiveChange("CONSTRAINT_RELAXED", field);
}
```

#### Rule 3: Enum Value Added

```typescript
// ADDITIVE: Enum value added
oldSchema: {
  properties: {
    status: { type: "string", enum: ["active", "inactive"] }
  }
}

newSchema: {
  properties: {
    status: { type: "string", enum: ["active", "inactive", "pending"] } // "pending" added
  }
}

// Detection
if (newEnumValues.includes(value) && !oldEnumValues.includes(value)) {
  additiveChange("ENUM_VALUE_ADDED", field);
}
```

---

## Compatibility Rules

### Type Coercion Rules

Some type changes are coercible (non-breaking):

```typescript
function isCoercible(oldType: string, newType: string): boolean {
  const coercions: Record<string, string[]> = {
    "integer": ["number"], // integer → number is safe
    "number": [],          // number → integer is NOT safe
    "string": [],          // string → anything is NOT safe
    "boolean": []          // boolean → anything is NOT safe
  };

  return coercions[oldType]?.includes(newType) || false;
}

// Examples
isCoercible("integer", "number"); // true (widening)
isCoercible("number", "integer"); // false (narrowing)
isCoercible("string", "integer"); // false (incompatible)
```

### Constraint Comparison

**Tightening (Breaking)**:
- Increase minimum
- Decrease maximum
- Decrease maxLength
- Increase minLength
- More restrictive pattern

**Relaxing (Additive)**:
- Decrease minimum
- Increase maximum
- Increase maxLength
- Decrease minLength
- Less restrictive pattern

```typescript
function isConstraintTightened(
  constraint: string,
  oldValue: any,
  newValue: any
): boolean {
  switch (constraint) {
    case "minimum":
    case "minLength":
    case "minItems":
      return newValue > oldValue; // Increase = tightening

    case "maximum":
    case "maxLength":
    case "maxItems":
      return newValue < oldValue; // Decrease = tightening

    default:
      return false;
  }
}
```

---

## Edge Cases

### Edge Case 1: Nested Field Changes

**Scenario**: Change in deeply nested object structure.

**Handling**: Recursively analyze nested fields:

```typescript
async function analyzeNestedFields(
  oldSchema: object,
  newSchema: object,
  path: string = ""
): Promise<Change[]> {
  const changes: Change[] = [];

  // Analyze current level
  for (const field of Object.keys(oldSchema.properties || {})) {
    const fieldPath = path ? `${path}.${field}` : field;
    const oldField = oldSchema.properties[field];
    const newField = newSchema.properties?.[field];

    if (!newField) {
      changes.push({
        type: "FIELD_REMOVED",
        field: fieldPath,
        severity: "BREAKING"
      });
    } else if (oldField.type === "object" && newField.type === "object") {
      // Recurse into nested object
      const nestedChanges = await analyzeNestedFields(oldField, newField, fieldPath);
      changes.push(...nestedChanges);
    } else if (oldField.type !== newField.type) {
      changes.push({
        type: "TYPE_CHANGED",
        field: fieldPath,
        old_type: oldField.type,
        new_type: newField.type,
        severity: "BREAKING"
      });
    }
  }

  return changes;
}
```

---

### Edge Case 2: Array Item Schema Changes

**Scenario**: Schema for array items changed.

**Handling**: Analyze item schema separately:

```typescript
// Old schema
{
  type: "array",
  items: {
    type: "object",
    properties: {
      id: { type: "string" }
    }
  }
}

// New schema
{
  type: "array",
  items: {
    type: "object",
    properties: {
      id: { type: "string" },
      name: { type: "string" } // Added field
    },
    required: ["id", "name"] // name now required
  }
}

// Detection
if (schema.type === "array" && schema.items) {
  const itemChanges = await checker.check(oldSchema.items, newSchema.items);

  if (itemChanges.breaking_changes.length > 0) {
    breakingChange("ARRAY_ITEM_SCHEMA_BREAKING", field);
  }
}
```

---

### Edge Case 3: Pattern Regex Comparison

**Scenario**: Pattern regex changed - determine if more or less restrictive.

**Handling**: Conservative approach - assume breaking unless provably relaxed:

```typescript
function isPatternRelaxed(oldPattern: string, newPattern: string): boolean {
  // Conservative: If patterns differ, assume breaking unless proven safe

  // Special case: Adding alternatives is relaxing
  // Old: "^[A-Z]+$"
  // New: "^[A-Z]+$|^[0-9]+$" (added number alternative)
  if (newPattern.includes(oldPattern)) {
    return true;
  }

  // Special case: Removing exact match requirement
  // Old: "^exact$"
  // New: "exact" (allows substring match)
  if (oldPattern.startsWith("^") && oldPattern.endsWith("$") &&
      newPattern === oldPattern.substring(1, oldPattern.length - 1)) {
    return true;
  }

  // Default: Assume breaking
  return false;
}
```

---

### Edge Case 4: OneOf/AnyOf Schema Changes

**Scenario**: Schema uses `oneOf` or `anyOf` for polymorphism.

**Handling**: Analyze each variant separately:

```typescript
// Old schema
{
  oneOf: [
    { type: "string" },
    { type: "integer" }
  ]
}

// New schema
{
  oneOf: [
    { type: "string" },
    { type: "integer" },
    { type: "boolean" } // Added variant
  ]
}

// Detection
function analyzeOneOf(oldOneOf: any[], newOneOf: any[]): Change[] {
  const changes: Change[] = [];

  // Removing variant = BREAKING
  for (const oldVariant of oldOneOf) {
    const exists = newOneOf.some(newVariant =>
      isSchemaEqual(oldVariant, newVariant)
    );

    if (!exists) {
      changes.push({
        type: "ONEOF_VARIANT_REMOVED",
        severity: "BREAKING"
      });
    }
  }

  // Adding variant = ADDITIVE
  for (const newVariant of newOneOf) {
    const exists = oldOneOf.some(oldVariant =>
      isSchemaEqual(oldVariant, newVariant)
    );

    if (!exists) {
      changes.push({
        type: "ONEOF_VARIANT_ADDED",
        severity: "ADDITIVE"
      });
    }
  }

  return changes;
}
```

---

### Edge Case 5: Default Value Changes

**Scenario**: Default value for field changed.

**Handling**: Classify as non-functional (doesn't affect validation):

```typescript
// Old schema
{
  properties: {
    status: { type: "string", default: "pending" }
  }
}

// New schema
{
  properties: {
    status: { type: "string", default: "active" } // Changed default
  }
}

// Detection
if (oldField.default !== newField.default) {
  nonFunctionalChange("DEFAULT_VALUE_CHANGED", field);
  // Not breaking (only affects newly created records)
}
```

---

### Edge Case 6: Format Attribute Changes

**Scenario**: Format attribute changed (e.g., email, uri, date-time).

**Handling**: Tightening format is breaking:

```typescript
// Old schema
{
  properties: {
    url: { type: "string" } // No format constraint
  }
}

// New schema
{
  properties: {
    url: { type: "string", format: "uri" } // Format added
  }
}

// Detection
if (!oldField.format && newField.format) {
  breakingChange("FORMAT_CONSTRAINT_ADDED", field);
  // Now requires URI format, old data may not comply
}

if (oldField.format && !newField.format) {
  additiveChange("FORMAT_CONSTRAINT_REMOVED", field);
  // Format constraint relaxed
}
```

---

### Edge Case 7: Read-Only Field Changes

**Scenario**: Field marked as `readOnly` or `writeOnly` changed.

**Handling**: Adding readOnly is breaking for writes:

```typescript
// Old schema
{
  properties: {
    created_at: { type: "string" }
  }
}

// New schema
{
  properties: {
    created_at: { type: "string", readOnly: true } // Now read-only
  }
}

// Detection
if (!oldField.readOnly && newField.readOnly) {
  breakingChange("FIELD_MADE_READONLY", field);
  // Consumers can no longer write to this field
}
```

---

## Performance Characteristics

### Latency Targets

| Operation | Input Size | Target Latency (p95) | Notes |
|-----------|-----------|---------------------|-------|
| `check()` | 2 simple schemas | < 10ms | Shallow comparison |
| `check()` | 2 complex schemas (100 fields) | < 100ms | Deep comparison |
| `detectBreakingChanges()` | 2 schemas | < 50ms | Focused analysis |
| `detectAdditiveChanges()` | 2 schemas | < 50ms | Focused analysis |
| `generateReport()` | 2 schemas | < 200ms | Full analysis + report generation |
| `validateEvolutionPath()` | 10 versions | < 500ms | Multiple comparisons |

### Throughput

- **Comparisons/second**: 10,000+ (simple schemas)
- **Comparisons/second**: 1,000-10,000 (complex schemas)

### Scalability

| Schema Complexity | Performance | Notes |
|------------------|-------------|-------|
| < 10 fields | Excellent (< 10ms) | Fast |
| 10-100 fields | Good (< 100ms) | Acceptable |
| 100-1000 fields | Moderate (< 1s) | Consider caching |
| > 1000 fields | Slower (< 5s) | Optimize comparison |

### Optimization Tips

**1. Cache Schema Comparisons**

```typescript
class CachedCompatibilityChecker {
  private cache = new Map<string, CompatibilityReport>();

  async check(oldSchema: object, newSchema: object): Promise<CompatibilityReport> {
    const key = `${hash(oldSchema)}-${hash(newSchema)}`;

    if (this.cache.has(key)) {
      return this.cache.get(key)!;
    }

    const report = await this.doCheck(oldSchema, newSchema);
    this.cache.set(key, report);

    return report;
  }
}
```

**2. Parallelize Field Comparisons**

```typescript
async function checkFieldsInParallel(
  oldSchema: object,
  newSchema: object
): Promise<Change[]> {
  const fields = Object.keys(oldSchema.properties || {});

  // Compare fields in parallel
  const promises = fields.map(field =>
    compareField(oldSchema.properties[field], newSchema.properties?.[field], field)
  );

  const results = await Promise.all(promises);

  return results.flat();
}
```

---

## Implementation Notes

### JavaScript/TypeScript Implementation

```typescript
export class BackwardCompatibilityChecker {
  async check(oldSchema: object, newSchema: object): Promise<CompatibilityReport> {
    const breaking_changes = await this.detectBreakingChanges(oldSchema, newSchema);
    const additive_changes = await this.detectAdditiveChanges(oldSchema, newSchema);
    const non_functional_changes = this.detectNonFunctionalChanges(oldSchema, newSchema);

    const compatible = breaking_changes.length === 0;

    let recommendation: string;
    if (breaking_changes.length > 0) {
      recommendation = "MAJOR version bump (breaking changes detected)";
    } else if (additive_changes.length > 0) {
      recommendation = "MINOR version bump (backward compatible additions)";
    } else {
      recommendation = "PATCH version bump (non-functional changes only)";
    }

    const risk_level = breaking_changes.length > 0 ? "HIGH" :
                       additive_changes.length > 5 ? "MEDIUM" : "LOW";

    return {
      compatible,
      breaking_changes,
      additive_changes,
      non_functional_changes,
      recommendation,
      risk_level
    };
  }

  async detectBreakingChanges(
    oldSchema: object,
    newSchema: object
  ): Promise<BreakingChange[]> {
    const changes: BreakingChange[] = [];

    const oldProps = oldSchema.properties || {};
    const newProps = newSchema.properties || {};

    // Check for removed fields
    for (const field of Object.keys(oldProps)) {
      if (!newProps[field]) {
        changes.push({
          type: "FIELD_REMOVED",
          field,
          severity: "BREAKING",
          description: `Field '${field}' removed`,
          impact: "Consumers expecting this field will fail",
          recommendation: "MAJOR version bump required"
        });
      }
    }

    // Check for type changes
    for (const field of Object.keys(oldProps)) {
      const oldField = oldProps[field];
      const newField = newProps[field];

      if (newField && oldField.type !== newField.type) {
        if (!this.isCoercible(oldField.type, newField.type)) {
          changes.push({
            type: "TYPE_CHANGED",
            field,
            old_value: oldField.type,
            new_value: newField.type,
            severity: "BREAKING",
            description: `Field type changed from ${oldField.type} to ${newField.type}`,
            impact: "Old data with old type will fail validation",
            recommendation: "MAJOR version bump required"
          });
        }
      }
    }

    // Check for new required fields
    const oldRequired = oldSchema.required || [];
    const newRequired = newSchema.required || [];

    for (const field of newRequired) {
      if (!oldRequired.includes(field)) {
        changes.push({
          type: "FIELD_REQUIRED_ADDED",
          field,
          severity: "BREAKING",
          description: `Field '${field}' is now required`,
          impact: "Old data without this field will fail validation",
          recommendation: "MAJOR version bump required"
        });
      }
    }

    // Check for tightened constraints
    const constraintChanges = await this.detectConstraintChanges(oldSchema, newSchema);
    const tightened = constraintChanges.filter(c => c.breaking);

    for (const change of tightened) {
      changes.push({
        type: "CONSTRAINT_TIGHTENED",
        field: change.field,
        severity: "BREAKING",
        description: change.description,
        impact: "Old data outside new constraint will fail validation",
        recommendation: "MAJOR version bump required",
        old_value: change.old_value,
        new_value: change.new_value
      });
    }

    return changes;
  }

  async detectAdditiveChanges(
    oldSchema: object,
    newSchema: object
  ): Promise<AdditiveChange[]> {
    const changes: AdditiveChange[] = [];

    const oldProps = oldSchema.properties || {};
    const newProps = newSchema.properties || {};

    // Check for added optional fields
    for (const field of Object.keys(newProps)) {
      if (!oldProps[field]) {
        const isRequired = (newSchema.required || []).includes(field);

        if (!isRequired) {
          changes.push({
            type: "FIELD_ADDED",
            field,
            severity: "ADDITIVE",
            description: `Optional field '${field}' added`
          });
        }
      }
    }

    // Check for relaxed constraints
    const constraintChanges = await this.detectConstraintChanges(oldSchema, newSchema);
    const relaxed = constraintChanges.filter(c => !c.breaking);

    for (const change of relaxed) {
      changes.push({
        type: "CONSTRAINT_RELAXED",
        field: change.field,
        severity: "ADDITIVE",
        description: change.description,
        old_value: change.old_value,
        new_value: change.new_value
      });
    }

    return changes;
  }

  async detectConstraintChanges(
    oldSchema: object,
    newSchema: object
  ): Promise<ConstraintChange[]> {
    const changes: ConstraintChange[] = [];

    const oldProps = oldSchema.properties || {};
    const newProps = newSchema.properties || {};

    for (const field of Object.keys(oldProps)) {
      const oldField = oldProps[field];
      const newField = newProps[field];

      if (!newField) continue;

      // Check minimum
      if (oldField.minimum !== undefined && newField.minimum !== undefined) {
        if (newField.minimum > oldField.minimum) {
          changes.push({
            field,
            constraint: "minimum",
            old_value: oldField.minimum,
            new_value: newField.minimum,
            breaking: true,
            description: `Minimum increased from ${oldField.minimum} to ${newField.minimum}`
          });
        } else if (newField.minimum < oldField.minimum) {
          changes.push({
            field,
            constraint: "minimum",
            old_value: oldField.minimum,
            new_value: newField.minimum,
            breaking: false,
            description: `Minimum decreased from ${oldField.minimum} to ${newField.minimum}`
          });
        }
      }

      // Check maximum
      if (oldField.maximum !== undefined && newField.maximum !== undefined) {
        if (newField.maximum < oldField.maximum) {
          changes.push({
            field,
            constraint: "maximum",
            old_value: oldField.maximum,
            new_value: newField.maximum,
            breaking: true,
            description: `Maximum decreased from ${oldField.maximum} to ${newField.maximum}`
          });
        } else if (newField.maximum > oldField.maximum) {
          changes.push({
            field,
            constraint: "maximum",
            old_value: oldField.maximum,
            new_value: newField.maximum,
            breaking: false,
            description: `Maximum increased from ${oldField.maximum} to ${newField.maximum}`
          });
        }
      }

      // Similar checks for minLength, maxLength, pattern, etc.
    }

    return changes;
  }

  private isCoercible(oldType: string, newType: string): boolean {
    const coercions: Record<string, string[]> = {
      "integer": ["number"],
      "number": [],
      "string": [],
      "boolean": []
    };

    return coercions[oldType]?.includes(newType) || false;
  }

  private detectNonFunctionalChanges(oldSchema: object, newSchema: object): Change[] {
    const changes: Change[] = [];

    // Check description changes
    if (oldSchema.description !== newSchema.description) {
      changes.push({
        type: "DESCRIPTION_CHANGED",
        severity: "NON_FUNCTIONAL",
        description: "Schema description updated"
      });
    }

    return changes;
  }
}
```

---

## Security Considerations

### 1. Schema Validation

Validate schemas before comparison:

```typescript
async function check(oldSchema: object, newSchema: object): Promise<CompatibilityReport> {
  // Validate both schemas
  validateJsonSchema(oldSchema);
  validateJsonSchema(newSchema);

  // Proceed with comparison
  return this.doCheck(oldSchema, newSchema);
}
```

### 2. Prevent Denial of Service

Limit schema complexity to prevent DoS:

```typescript
function validateSchemaComplexity(schema: object): void {
  const maxDepth = 10;
  const maxProperties = 1000;

  const depth = calculateDepth(schema);
  const properties = countProperties(schema);

  if (depth > maxDepth) {
    throw new Error(`Schema depth exceeds maximum (${maxDepth})`);
  }

  if (properties > maxProperties) {
    throw new Error(`Schema properties exceed maximum (${maxProperties})`);
  }
}
```

---

## Integration Patterns

### Pattern 1: Pre-Deployment Validation

```typescript
// CI/CD pipeline: Validate schema changes
async function validateSchemaChange(
  schemaId: string,
  newSchemaVersion: string
): Promise<void> {
  const checker = new BackwardCompatibilityChecker();
  const registry = new SchemaRegistry();

  // Get current version
  const current = await registry.getLatest(schemaId);
  const proposed = await registry.getVersion(schemaId, newSchemaVersion);

  // Check compatibility
  const report = await checker.check(current.schema, proposed.schema);

  if (report.breaking_changes.length > 0) {
    console.error("Breaking changes detected:");
    for (const change of report.breaking_changes) {
      console.error(`- ${change.description}`);
    }

    throw new Error("Deployment blocked due to breaking changes");
  }

  console.log("✓ Schema change is backward compatible");
}
```

### Pattern 2: Automatic Version Bump

```typescript
// Auto-determine version bump based on compatibility
async function autoVersionBump(
  oldSchema: object,
  newSchema: object,
  currentVersion: string
): Promise<string> {
  const checker = new BackwardCompatibilityChecker();
  const manager = new SchemaVersionManager();

  const report = await checker.check(oldSchema, newSchema);

  let bumpType: BumpType;
  if (report.breaking_changes.length > 0) {
    bumpType = "MAJOR";
  } else if (report.additive_changes.length > 0) {
    bumpType = "MINOR";
  } else {
    bumpType = "PATCH";
  }

  return manager.bumpVersion(currentVersion, bumpType);
}
```

---

## Multi-Domain Examples

(Already covered in "Multi-Domain Applicability" section with 7 domains: Finance, Healthcare, Legal, Research, E-commerce, SaaS, Insurance)

---

## Testing Strategy

### Unit Tests

```typescript
describe('BackwardCompatibilityChecker', () => {
  let checker: BackwardCompatibilityChecker;

  beforeEach(() => {
    checker = new BackwardCompatibilityChecker();
  });

  describe('detectBreakingChanges()', () => {
    it('should detect removed field', async () => {
      const oldSchema = {
        type: 'object',
        properties: {
          id: { type: 'string' },
          legacy_id: { type: 'string' }
        }
      };

      const newSchema = {
        type: 'object',
        properties: {
          id: { type: 'string' }
        }
      };

      const changes = await checker.detectBreakingChanges(oldSchema, newSchema);

      expect(changes).toHaveLength(1);
      expect(changes[0].type).toBe('FIELD_REMOVED');
      expect(changes[0].field).toBe('legacy_id');
    });

    it('should detect type change', async () => {
      const oldSchema = {
        type: 'object',
        properties: {
          age: { type: 'string' }
        }
      };

      const newSchema = {
        type: 'object',
        properties: {
          age: { type: 'integer' }
        }
      };

      const changes = await checker.detectBreakingChanges(oldSchema, newSchema);

      expect(changes).toHaveLength(1);
      expect(changes[0].type).toBe('TYPE_CHANGED');
      expect(changes[0].old_value).toBe('string');
      expect(changes[0].new_value).toBe('integer');
    });

    it('should detect new required field', async () => {
      const oldSchema = {
        type: 'object',
        properties: {
          id: { type: 'string' },
          email: { type: 'string' }
        },
        required: ['id']
      };

      const newSchema = {
        type: 'object',
        properties: {
          id: { type: 'string' },
          email: { type: 'string' }
        },
        required: ['id', 'email']
      };

      const changes = await checker.detectBreakingChanges(oldSchema, newSchema);

      expect(changes).toHaveLength(1);
      expect(changes[0].type).toBe('FIELD_REQUIRED_ADDED');
      expect(changes[0].field).toBe('email');
    });
  });

  describe('detectAdditiveChanges()', () => {
    it('should detect added optional field', async () => {
      const oldSchema = {
        type: 'object',
        properties: {
          id: { type: 'string' }
        }
      };

      const newSchema = {
        type: 'object',
        properties: {
          id: { type: 'string' },
          email: { type: 'string' }
        }
      };

      const changes = await checker.detectAdditiveChanges(oldSchema, newSchema);

      expect(changes).toHaveLength(1);
      expect(changes[0].type).toBe('FIELD_ADDED');
      expect(changes[0].field).toBe('email');
    });

    it('should detect relaxed constraint', async () => {
      const oldSchema = {
        type: 'object',
        properties: {
          age: { type: 'integer', minimum: 18 }
        }
      };

      const newSchema = {
        type: 'object',
        properties: {
          age: { type: 'integer', minimum: 0 }
        }
      };

      const changes = await checker.detectAdditiveChanges(oldSchema, newSchema);

      expect(changes).toHaveLength(1);
      expect(changes[0].type).toBe('CONSTRAINT_RELAXED');
    });
  });

  describe('check()', () => {
    it('should return compatible for backward compatible changes', async () => {
      const oldSchema = {
        type: 'object',
        properties: {
          id: { type: 'string' }
        }
      };

      const newSchema = {
        type: 'object',
        properties: {
          id: { type: 'string' },
          email: { type: 'string' }
        }
      };

      const report = await checker.check(oldSchema, newSchema);

      expect(report.compatible).toBe(true);
      expect(report.breaking_changes).toHaveLength(0);
      expect(report.additive_changes.length).toBeGreaterThan(0);
    });

    it('should return incompatible for breaking changes', async () => {
      const oldSchema = {
        type: 'object',
        properties: {
          id: { type: 'string' },
          name: { type: 'string' }
        }
      };

      const newSchema = {
        type: 'object',
        properties: {
          id: { type: 'string' }
        }
      };

      const report = await checker.check(oldSchema, newSchema);

      expect(report.compatible).toBe(false);
      expect(report.breaking_changes.length).toBeGreaterThan(0);
    });
  });
});
```

---

## Migration Guide

### From Manual Compatibility Checks to BackwardCompatibilityChecker

```typescript
// Before: Manual review
// Developer reviews PR and manually checks for breaking changes
// Error-prone, time-consuming

// After: Automated checking
const checker = new BackwardCompatibilityChecker();

const report = await checker.check(oldSchema, newSchema);

if (report.breaking_changes.length > 0) {
  console.error("❌ Breaking changes detected:");
  for (const change of report.breaking_changes) {
    console.error(`   ${change.description}`);
  }
  process.exit(1);
} else {
  console.log("✅ No breaking changes detected");
}
```

---

## Related Primitives

- **SchemaRegistry**: Uses compatibility checker for version validation
- **SchemaVersionManager**: Uses breaking change detection for version bump calculation
- **MigrationEngine**: Uses compatibility reports for migration plan generation
- **ParserSelector**: Similar compatibility logic for parser versions

---

## References

- [JSON Schema Specification](https://json-schema.org/)
- [Semantic Versioning](https://semver.org/)
- [Schema Evolution Best Practices](https://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html)
- [Objective Layer Architecture](../architecture/objective-layer.md)
- [Schema Registry Vertical 5.2](../verticals/schema-registry.md)

---

**End of BackwardCompatibilityChecker OL Primitive Specification**
