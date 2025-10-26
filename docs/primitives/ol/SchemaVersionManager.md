# SchemaVersionManager OL Primitive

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
8. [Version Comparison Logic](#version-comparison-logic)
9. [Breaking Change Detection](#breaking-change-detection)
10. [Version Upgrade Paths](#version-upgrade-paths)
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

The **SchemaVersionManager** is a semantic versioning engine that manages version numbers, detects breaking changes, and calculates upgrade/downgrade paths for schema evolution. It implements strict [semver](https://semver.org/) rules to ensure predictable schema evolution and compatibility.

### Key Capabilities

- **Semantic Versioning**: Enforces major.minor.patch versioning rules
- **Breaking Change Detection**: Analyzes schema diffs to determine version bump type
- **Version Comparison**: Efficient comparison and sorting of semantic versions
- **Upgrade Path Calculation**: Determines safe upgrade paths between versions
- **Version Parsing**: Extracts major, minor, patch components from version strings
- **Range Checking**: Validates if version satisfies version ranges (e.g., "^1.2.0", "~1.2.3")

### Design Philosophy

The SchemaVersionManager follows four core principles:

1. **Strict Semver Compliance**: Follows semver 2.0.0 specification exactly
2. **Deterministic Bumps**: Same changes always result in same version bump type
3. **Breaking Change Safety**: Breaking changes MUST result in major version bump
4. **Upgrade Path Transparency**: Users know exactly what changes between versions

---

## Purpose & Scope

### Problem Statement

In schema evolution, version management poses critical challenges:

- **Ambiguous Versioning**: Developers unsure whether change requires major, minor, or patch bump
- **Breaking Changes Undetected**: Schema changes break consumers without warning
- **No Upgrade Guidance**: Consumers don't know which versions are compatible
- **Manual Version Assignment**: Error-prone manual version number selection
- **No Downgrade Safety**: Unclear if downgrading to older version is safe

Traditional approaches have significant limitations:

**Approach 1: Manual Versioning**
- ❌ Human error in version number selection
- ❌ Inconsistent versioning across schemas
- ❌ No automated breaking change detection

**Approach 2: Automatic Timestamp Versioning**
- ❌ No semantic meaning in version numbers
- ❌ Can't determine compatibility from version
- ❌ No distinction between breaking and non-breaking changes

**Approach 3: SchemaVersionManager**
- ✅ Automated breaking change detection
- ✅ Consistent semver-based versioning
- ✅ Upgrade path calculation
- ✅ Version range validation
- ✅ Predictable schema evolution

### Solution

The SchemaVersionManager implements **semantic versioning automation** with:

1. **Version Parsing**: Extract major, minor, patch from version strings
2. **Version Comparison**: Compare versions using semver rules (1.2.3 < 1.10.0)
3. **Breaking Change Detection**: Analyze schema diffs to determine bump type
4. **Version Bumping**: Automatically calculate next version based on changes
5. **Range Validation**: Check if version satisfies constraints (^1.2.0, ~1.2.3)

---

## Multi-Domain Applicability

The SchemaVersionManager is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Manage financial transaction schema versions across payment systems.

**Versioning Scenarios**:
- **PATCH**: Fix typo in merchant field description (1.0.0 → 1.0.1)
- **MINOR**: Add optional `merchant_category_code` field (1.0.1 → 1.1.0)
- **MAJOR**: Change `amount` from string to number (1.1.0 → 2.0.0)

**Example**:
```typescript
const manager = new SchemaVersionManager();

// Current version
const currentVersion = "1.0.0";

// Proposed schema change: add optional field
const changeType = await manager.detectChangeType(
  oldSchema, // v1.0.0
  newSchema  // v1.0.0 + optional field
);

console.log(changeType); // "MINOR"

// Calculate next version
const nextVersion = await manager.bumpVersion(currentVersion, "MINOR");
console.log(nextVersion); // "1.1.0"

// Verify upgrade is safe
const isBackwardCompatible = await manager.isBackwardCompatible("1.0.0", "1.1.0");
console.log(isBackwardCompatible); // true (MINOR is always backward compatible)
```

**Compliance**: SOX requires deterministic versioning for audit trails.

---

### 2. Healthcare

**Use Case**: Version FHIR patient record schemas with strict compatibility guarantees.

**Versioning Scenarios**:
- **PATCH**: Update description of `birthDate` field (4.0.0 → 4.0.1)
- **MINOR**: Add optional `race` field for demographics (4.0.1 → 4.1.0)
- **MAJOR**: Make `gender` field required (4.1.0 → 5.0.0)

**Example**:
```typescript
// Detect breaking change: add required field
const changes = await manager.detectBreakingChanges(
  fhirPatient_v4_1_0,
  fhirPatient_v5_0_0_draft
);

console.log(changes);
// [
//   {
//     type: "FIELD_REQUIRED_ADDED",
//     field: "gender",
//     severity: "BREAKING",
//     description: "Field 'gender' is now required"
//   }
// ]

// Calculate required version bump
const requiredBump = manager.calculateRequiredBump(changes);
console.log(requiredBump); // "MAJOR" (breaking change detected)

// Bump version
const nextVersion = manager.bumpVersion("4.1.0", requiredBump);
console.log(nextVersion); // "5.0.0"
```

**Compliance**: HIPAA requires version control for PHI data structures.

---

### 3. Legal

**Use Case**: Version legal document schemas for e-discovery systems.

**Versioning Scenarios**:
- **PATCH**: Fix regex pattern in case_number field (1.2.0 → 1.2.1)
- **MINOR**: Add optional `exhibits` array field (1.2.1 → 1.3.0)
- **MAJOR**: Remove deprecated `legacy_id` field (1.3.0 → 2.0.0)

**Example**:
```typescript
// Compare two versions
const comparison = await manager.compareVersions("1.2.1", "2.0.0");

console.log(comparison);
// {
//   result: -1, // 1.2.1 < 2.0.0
//   diff: {
//     major: 1,  // Major version increased by 1
//     minor: -2, // Minor version decreased (reset to 0)
//     patch: -1  // Patch version decreased (reset to 0)
//   },
//   isUpgrade: true,
//   isBreaking: true // Major version change = breaking
// }

// Check if version satisfies range
const satisfiesRange = manager.satisfiesRange("1.2.1", "^1.0.0");
console.log(satisfiesRange); // true (1.2.1 is compatible with ^1.0.0)
```

---

### 4. Research (RSRCH - Utilitario)

**Use Case**: Version founder fact schemas for RSRCH utilitario research system.

**Versioning Scenarios**:
- **PATCH**: Correct example in fact schema documentation (1.0.0 → 1.0.1)
- **MINOR**: Add optional `confidence` field for fact credibility (1.0.1 → 1.1.0)
- **MAJOR**: Change `subject_entity` from string to structured object (1.1.0 → 2.0.0)

**Example**:
```typescript
// Parse version components
const parsed = manager.parseVersion("1.1.0");

console.log(parsed);
// {
//   major: 1,
//   minor: 1,
//   patch: 0,
//   original: "1.1.0"
// }

// Get upgrade path for fact schema evolution
const upgradePath = manager.getUpgradePath("1.0.0", "2.0.0");

console.log(upgradePath);
// [
//   { from: "1.0.0", to: "1.0.1", type: "PATCH" },  // Fix doc examples
//   { from: "1.0.1", to: "1.1.0", type: "MINOR" },  // Add confidence field
//   { from: "1.1.0", to: "2.0.0", type: "MAJOR" }   // Structured subject_entity
// ]
```

---

### 5. E-commerce

**Use Case**: Version product catalog schemas across marketplaces.

**Versioning Scenarios**:
- **PATCH**: Fix price field decimal precision (1.0.0 → 1.0.1)
- **MINOR**: Add optional `brand` field (1.0.1 → 1.1.0)
- **MAJOR**: Make `category` field required (1.1.0 → 2.0.0)

**Example**:
```typescript
// Determine if version is breaking change
const isBreaking = manager.isBreakingChange("1.5.0", "2.0.0");
console.log(isBreaking); // true (major version increased)

// Sort versions
const versions = ["1.10.0", "1.2.0", "2.0.0", "1.2.3", "1.9.0"];
const sorted = manager.sortVersions(versions, "desc");

console.log(sorted);
// ["2.0.0", "1.10.0", "1.9.0", "1.2.3", "1.2.0"]
```

---

### 6. SaaS

**Use Case**: Version API schemas for multi-tenant SaaS platforms.

**Versioning Scenarios**:
- **PATCH**: Update field description (1.0.0 → 1.0.1)
- **MINOR**: Add optional webhook field (1.0.1 → 1.1.0)
- **MAJOR**: Change authentication field type (1.1.0 → 2.0.0)

**Example**:
```typescript
// Check version compatibility
const compat = manager.checkCompatibility("1.5.0", "1.8.0");

console.log(compat);
// {
//   isCompatible: true, // Same major version
//   canAutoUpgrade: true, // Minor/patch only
//   requiresManualMigration: false,
//   changeType: "MINOR",
//   recommendation: "Auto-upgrade recommended"
// }

// Check if version is latest in major version
const isLatestInMajor = manager.isLatestInMajorVersion("1.5.0", [
  "1.2.0", "1.5.0", "1.8.0", "2.0.0"
]);
console.log(isLatestInMajor); // false (1.8.0 is latest in v1)
```

---

### 7. Insurance

**Use Case**: Version insurance policy schemas across underwriting systems.

**Versioning Scenarios**:
- **PATCH**: Fix typo in coverage_type enum (1.0.0 → 1.0.1)
- **MINOR**: Add optional `deductible` field (1.0.1 → 1.1.0)
- **MAJOR**: Remove deprecated `legacy_policy_number` field (1.1.0 → 2.0.0)

**Example**:
```typescript
// Validate version format
const isValid = manager.isValidVersion("1.2.3");
console.log(isValid); // true

const isInvalid = manager.isValidVersion("1.2");
console.log(isInvalid); // false (missing patch)

// Get next available version
const nextAvailable = manager.getNextAvailableVersion("1.5.0", [
  "1.5.0", "1.5.1", "1.6.0"
], "PATCH");
console.log(nextAvailable); // "1.5.2" (1.5.1 exists, so next patch is 1.5.2)
```

---

### Cross-Domain Benefits

**Consistent Versioning**: All domains benefit from:
- Deterministic version number assignment
- Breaking change detection before deployment
- Upgrade path calculation for migrations
- Version range validation for compatibility

**Regulatory Compliance**: Supports:
- SOX (Finance): Audit trail of schema versions
- HIPAA (Healthcare): Data structure versioning
- GDPR (Privacy): Schema evolution tracking
- Legal Discovery: Version history preservation

---

## Core Concepts

### Semantic Versioning (Semver)

Version format: **MAJOR.MINOR.PATCH**

**MAJOR**: Incremented for breaking changes
- Remove field
- Rename field
- Change field type
- Add required field
- Tighten constraint (e.g., increase minimum)

**MINOR**: Incremented for backward-compatible additions
- Add optional field
- Relax constraint (e.g., decrease minimum)
- Add enum value
- Add new schema definition

**PATCH**: Incremented for bug fixes and non-functional changes
- Fix typo in description
- Update documentation
- Correct example
- Fix regex pattern (if logically equivalent)

### Version Comparison

Versions are compared lexicographically by major, then minor, then patch:

```
1.0.0 < 1.0.1 < 1.1.0 < 1.10.0 < 2.0.0
```

**NOT alphabetically**:
```
❌ "1.10.0" < "1.2.0" (string comparison)
✅ 1.2.0 < 1.10.0 (semantic comparison)
```

### Version Ranges

**Caret (^)**: Compatible with same major version
- `^1.2.3` matches `>=1.2.3 <2.0.0`
- Allows MINOR and PATCH updates

**Tilde (~)**: Compatible with same minor version
- `~1.2.3` matches `>=1.2.3 <1.3.0`
- Allows PATCH updates only

**Exact**: Exact version match
- `1.2.3` matches exactly `1.2.3`

### Breaking Changes

A change is **breaking** if:
1. Existing valid data becomes invalid
2. Consumers must update code to handle change
3. Major version bump required

**Examples of breaking changes**:
- Remove field: Data with that field becomes invalid
- Add required field: Old data missing that field becomes invalid
- Change type: Data with old type becomes invalid
- Tighten constraint: Data outside new constraint becomes invalid

---

## Interface Definition

### TypeScript Interface

```typescript
interface SchemaVersionManager {
  /**
   * Parse a semantic version string into components
   *
   * @param version - Semantic version string (e.g., "1.2.3")
   * @returns Parsed version object with major, minor, patch
   * @throws ValidationError if version is invalid
   */
  parseVersion(version: string): ParsedVersion;

  /**
   * Compare two semantic versions
   *
   * @param v1 - First version
   * @param v2 - Second version
   * @returns -1 if v1 < v2, 0 if v1 == v2, 1 if v1 > v2
   */
  compareVersions(v1: string, v2: string): number;

  /**
   * Bump a version by major, minor, or patch
   *
   * @param version - Current version
   * @param bumpType - Type of bump (MAJOR, MINOR, PATCH)
   * @returns New version after bump
   */
  bumpVersion(version: string, bumpType: BumpType): string;

  /**
   * Detect if a schema change is a breaking change
   *
   * @param oldSchema - Previous schema
   * @param newSchema - New schema
   * @returns true if breaking change detected
   */
  isBreakingChange(oldSchema: object, newSchema: object): Promise<boolean>;

  /**
   * Calculate required version bump type based on schema changes
   *
   * @param oldSchema - Previous schema
   * @param newSchema - New schema
   * @returns Required bump type (MAJOR, MINOR, PATCH)
   */
  calculateRequiredBump(oldSchema: object, newSchema: object): Promise<BumpType>;

  /**
   * Check if version satisfies a version range
   *
   * @param version - Version to check
   * @param range - Version range (e.g., "^1.2.0", "~1.2.3", "1.2.3")
   * @returns true if version satisfies range
   */
  satisfiesRange(version: string, range: string): boolean;

  /**
   * Sort versions in ascending or descending order
   *
   * @param versions - Array of version strings
   * @param order - Sort order (asc or desc)
   * @returns Sorted array of versions
   */
  sortVersions(versions: string[], order?: 'asc' | 'desc'): string[];

  /**
   * Get upgrade path from one version to another
   *
   * @param fromVersion - Starting version
   * @param toVersion - Target version
   * @param availableVersions - List of published versions
   * @returns Array of version steps in upgrade path
   */
  getUpgradePath(
    fromVersion: string,
    toVersion: string,
    availableVersions: string[]
  ): VersionStep[];

  /**
   * Check if two versions are backward compatible
   *
   * @param oldVersion - Old version
   * @param newVersion - New version
   * @returns true if newVersion is backward compatible with oldVersion
   */
  isBackwardCompatible(oldVersion: string, newVersion: string): boolean;

  /**
   * Validate version string format
   *
   * @param version - Version string to validate
   * @returns true if valid semver format
   */
  isValidVersion(version: string): boolean;

  /**
   * Get latest version from a list
   *
   * @param versions - Array of version strings
   * @returns Latest (highest) version
   */
  getLatestVersion(versions: string[]): string | null;

  /**
   * Check compatibility between two versions
   *
   * @param v1 - First version
   * @param v2 - Second version
   * @returns Compatibility report
   */
  checkCompatibility(v1: string, v2: string): CompatibilityReport;
}
```

---

## Data Model

### ParsedVersion Type

```typescript
interface ParsedVersion {
  major: number;              // Major version number
  minor: number;              // Minor version number
  patch: number;              // Patch version number
  original: string;           // Original version string
}
```

### BumpType Enum

```typescript
type BumpType = 'MAJOR' | 'MINOR' | 'PATCH';
```

### VersionStep Type

```typescript
interface VersionStep {
  from: string;               // Source version
  to: string;                 // Target version
  type: BumpType;             // Type of version bump
  breaking: boolean;          // Whether step is breaking
}
```

### CompatibilityReport Type

```typescript
interface CompatibilityReport {
  isCompatible: boolean;              // Are versions compatible?
  canAutoUpgrade: boolean;            // Can upgrade without manual intervention?
  requiresManualMigration: boolean;   // Does upgrade require manual migration?
  changeType: BumpType | null;        // Type of change between versions
  recommendation: string;             // Human-readable recommendation
}
```

### VersionRange Type

```typescript
interface VersionRange {
  type: 'caret' | 'tilde' | 'exact'; // Range type
  baseVersion: string;                // Base version for range
  minVersion: string;                 // Minimum version (inclusive)
  maxVersion: string;                 // Maximum version (exclusive)
}
```

---

## Core Functionality

### 1. parseVersion()

Parse a semantic version string into components.

#### Signature

```typescript
parseVersion(version: string): ParsedVersion
```

#### Behavior

1. Validate format matches `major.minor.patch`
2. Extract numeric components
3. Return parsed object

#### Example

```typescript
const manager = new SchemaVersionManager();

const parsed = manager.parseVersion("1.2.3");

console.log(parsed);
// {
//   major: 1,
//   minor: 2,
//   patch: 3,
//   original: "1.2.3"
// }
```

#### Error Cases

```typescript
// Invalid format
try {
  manager.parseVersion("1.2"); // Missing patch
} catch (error) {
  console.log(error.message); // "Invalid semver format: must be major.minor.patch"
}

// Non-numeric components
try {
  manager.parseVersion("1.2.x");
} catch (error) {
  console.log(error.message); // "Invalid semver: patch must be numeric"
}
```

---

### 2. compareVersions()

Compare two semantic versions.

#### Signature

```typescript
compareVersions(v1: string, v2: string): number
```

#### Behavior

1. Parse both versions
2. Compare major, then minor, then patch
3. Return -1, 0, or 1

#### Example

```typescript
console.log(manager.compareVersions("1.0.0", "2.0.0")); // -1 (v1 < v2)
console.log(manager.compareVersions("1.10.0", "1.2.0")); // 1 (v1 > v2)
console.log(manager.compareVersions("1.2.3", "1.2.3")); // 0 (v1 == v2)

// NOT alphabetical comparison
console.log("1.10.0" < "1.2.0"); // true (string comparison - WRONG)
console.log(manager.compareVersions("1.10.0", "1.2.0")); // 1 (semantic - CORRECT)
```

---

### 3. bumpVersion()

Increment a version by major, minor, or patch.

#### Signature

```typescript
bumpVersion(version: string, bumpType: BumpType): string
```

#### Behavior

1. Parse current version
2. Increment appropriate component
3. Reset lower components to 0 (for major/minor bumps)
4. Return new version string

#### Example

```typescript
// Patch bump: increment patch, keep major/minor
console.log(manager.bumpVersion("1.2.3", "PATCH")); // "1.2.4"

// Minor bump: increment minor, reset patch
console.log(manager.bumpVersion("1.2.3", "MINOR")); // "1.3.0"

// Major bump: increment major, reset minor and patch
console.log(manager.bumpVersion("1.2.3", "MAJOR")); // "2.0.0"
```

#### Rules

```typescript
// PATCH: major.minor.patch → major.minor.(patch+1)
bumpVersion("1.2.3", "PATCH") // "1.2.4"

// MINOR: major.minor.patch → major.(minor+1).0
bumpVersion("1.2.3", "MINOR") // "1.3.0"

// MAJOR: major.minor.patch → (major+1).0.0
bumpVersion("1.2.3", "MAJOR") // "2.0.0"
```

---

### 4. isBreakingChange()

Detect if a schema change is breaking.

#### Signature

```typescript
isBreakingChange(oldSchema: object, newSchema: object): Promise<boolean>
```

#### Behavior

1. Compare schemas using BackwardCompatibilityChecker
2. Detect breaking changes
3. Return true if any breaking changes found

#### Example

```typescript
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
    name: { type: "string" }
  },
  required: ["id", "name"] // Added required field
};

const isBreaking = await manager.isBreakingChange(oldSchema, newSchema);
console.log(isBreaking); // true (added required field)
```

---

### 5. calculateRequiredBump()

Calculate required version bump based on schema changes.

#### Signature

```typescript
calculateRequiredBump(oldSchema: object, newSchema: object): Promise<BumpType>
```

#### Behavior

1. Analyze schema changes
2. Determine if breaking, additive, or non-functional
3. Return required bump type

#### Example

```typescript
// Scenario 1: Add optional field (MINOR)
const bump1 = await manager.calculateRequiredBump(
  { type: "object", properties: { id: { type: "string" } } },
  { type: "object", properties: { id: { type: "string" }, name: { type: "string" } } }
);
console.log(bump1); // "MINOR"

// Scenario 2: Change field type (MAJOR)
const bump2 = await manager.calculateRequiredBump(
  { type: "object", properties: { age: { type: "string" } } },
  { type: "object", properties: { age: { type: "number" } } }
);
console.log(bump2); // "MAJOR"

// Scenario 3: Fix description (PATCH)
const bump3 = await manager.calculateRequiredBump(
  { type: "object", description: "User data" },
  { type: "object", description: "User profile data" }
);
console.log(bump3); // "PATCH"
```

---

### 6. satisfiesRange()

Check if version satisfies a version range.

#### Signature

```typescript
satisfiesRange(version: string, range: string): boolean
```

#### Behavior

1. Parse range type (caret, tilde, exact)
2. Calculate min/max versions
3. Check if version falls within range

#### Example

```typescript
// Caret (^): compatible with same major version
console.log(manager.satisfiesRange("1.2.3", "^1.0.0")); // true
console.log(manager.satisfiesRange("1.5.0", "^1.0.0")); // true
console.log(manager.satisfiesRange("2.0.0", "^1.0.0")); // false

// Tilde (~): compatible with same minor version
console.log(manager.satisfiesRange("1.2.3", "~1.2.0")); // true
console.log(manager.satisfiesRange("1.2.9", "~1.2.0")); // true
console.log(manager.satisfiesRange("1.3.0", "~1.2.0")); // false

// Exact: exact match only
console.log(manager.satisfiesRange("1.2.3", "1.2.3")); // true
console.log(manager.satisfiesRange("1.2.4", "1.2.3")); // false
```

#### Range Semantics

```typescript
// ^1.2.3 means >=1.2.3 <2.0.0
// Compatible with all MINOR and PATCH updates in major version 1

// ~1.2.3 means >=1.2.3 <1.3.0
// Compatible with all PATCH updates in minor version 1.2

// 1.2.3 means exactly 1.2.3
// No compatibility, must match exactly
```

---

### 7. sortVersions()

Sort versions in ascending or descending order.

#### Signature

```typescript
sortVersions(versions: string[], order?: 'asc' | 'desc'): string[]
```

#### Behavior

1. Parse all versions
2. Sort using semantic comparison
3. Return sorted array

#### Example

```typescript
const versions = ["1.10.0", "1.2.0", "2.0.0", "1.2.3", "1.9.0"];

// Ascending (oldest first)
const asc = manager.sortVersions(versions, "asc");
console.log(asc);
// ["1.2.0", "1.2.3", "1.9.0", "1.10.0", "2.0.0"]

// Descending (newest first)
const desc = manager.sortVersions(versions, "desc");
console.log(desc);
// ["2.0.0", "1.10.0", "1.9.0", "1.2.3", "1.2.0"]
```

---

### 8. getUpgradePath()

Calculate upgrade path from one version to another.

#### Signature

```typescript
getUpgradePath(
  fromVersion: string,
  toVersion: string,
  availableVersions: string[]
): VersionStep[]
```

#### Behavior

1. Validate fromVersion and toVersion exist
2. Find all versions between from and to
3. Sort versions
4. Build upgrade path with version steps

#### Example

```typescript
const availableVersions = ["1.0.0", "1.1.0", "1.2.0", "2.0.0", "2.1.0"];

const path = manager.getUpgradePath("1.0.0", "2.1.0", availableVersions);

console.log(path);
// [
//   { from: "1.0.0", to: "1.1.0", type: "MINOR", breaking: false },
//   { from: "1.1.0", to: "1.2.0", type: "MINOR", breaking: false },
//   { from: "1.2.0", to: "2.0.0", type: "MAJOR", breaking: true },
//   { from: "2.0.0", to: "2.1.0", type: "MINOR", breaking: false }
// ]
```

---

### 9. isBackwardCompatible()

Check if two versions are backward compatible.

#### Signature

```typescript
isBackwardCompatible(oldVersion: string, newVersion: string): boolean
```

#### Behavior

1. Parse both versions
2. Check if major version is same
3. Return true if same major version (backward compatible)

#### Example

```typescript
// Same major version = backward compatible
console.log(manager.isBackwardCompatible("1.0.0", "1.5.0")); // true
console.log(manager.isBackwardCompatible("1.2.0", "1.2.1")); // true

// Different major version = NOT backward compatible
console.log(manager.isBackwardCompatible("1.5.0", "2.0.0")); // false
```

---

### 10. checkCompatibility()

Generate compatibility report between two versions.

#### Signature

```typescript
checkCompatibility(v1: string, v2: string): CompatibilityReport
```

#### Behavior

1. Compare versions
2. Determine change type
3. Generate recommendations

#### Example

```typescript
const report1 = manager.checkCompatibility("1.5.0", "1.8.0");

console.log(report1);
// {
//   isCompatible: true,
//   canAutoUpgrade: true,
//   requiresManualMigration: false,
//   changeType: "MINOR",
//   recommendation: "Auto-upgrade recommended (backward compatible)"
// }

const report2 = manager.checkCompatibility("1.5.0", "2.0.0");

console.log(report2);
// {
//   isCompatible: false,
//   canAutoUpgrade: false,
//   requiresManualMigration: true,
//   changeType: "MAJOR",
//   recommendation: "Manual migration required (breaking changes)"
// }
```

---

## Version Comparison Logic

### Comparison Algorithm

```typescript
function compareVersions(v1: string, v2: string): number {
  const p1 = parseVersion(v1);
  const p2 = parseVersion(v2);

  // Compare major
  if (p1.major !== p2.major) {
    return p1.major - p2.major;
  }

  // Compare minor
  if (p1.minor !== p2.minor) {
    return p1.minor - p2.minor;
  }

  // Compare patch
  return p1.patch - p2.patch;
}
```

### Examples

```typescript
// Major version difference
compareVersions("2.0.0", "1.9.9") // 1 (2.x > 1.x)

// Minor version difference (same major)
compareVersions("1.5.0", "1.10.0") // -1 (1.5.x < 1.10.x)

// Patch version difference (same major.minor)
compareVersions("1.2.5", "1.2.3") // 1 (1.2.5 > 1.2.3)

// Equal versions
compareVersions("1.2.3", "1.2.3") // 0
```

---

## Breaking Change Detection

### Breaking Change Rules

**Field Removed**:
```typescript
// BREAKING: Field 'legacy_id' removed
oldSchema: { properties: { id: {...}, legacy_id: {...} } }
newSchema: { properties: { id: {...} } }
// Required bump: MAJOR
```

**Field Type Changed**:
```typescript
// BREAKING: Type changed from string to number
oldSchema: { properties: { age: { type: "string" } } }
newSchema: { properties: { age: { type: "number" } } }
// Required bump: MAJOR
```

**Required Field Added**:
```typescript
// BREAKING: Field 'email' now required
oldSchema: { required: ["id"] }
newSchema: { required: ["id", "email"] }
// Required bump: MAJOR
```

**Constraint Tightened**:
```typescript
// BREAKING: Minimum increased from 0 to 18
oldSchema: { properties: { age: { type: "number", minimum: 0 } } }
newSchema: { properties: { age: { type: "number", minimum: 18 } } }
// Required bump: MAJOR
```

### Non-Breaking Changes

**Optional Field Added**:
```typescript
// NON-BREAKING: Optional field added
oldSchema: { properties: { id: {...} } }
newSchema: { properties: { id: {...}, email: {...} } }
// Required bump: MINOR
```

**Constraint Relaxed**:
```typescript
// NON-BREAKING: Minimum decreased from 18 to 0
oldSchema: { properties: { age: { type: "number", minimum: 18 } } }
newSchema: { properties: { age: { type: "number", minimum: 0 } } }
// Required bump: MINOR
```

**Description Updated**:
```typescript
// NON-BREAKING: Description changed
oldSchema: { description: "User data" }
newSchema: { description: "User profile data" }
// Required bump: PATCH
```

---

## Version Upgrade Paths

### Upgrade Path Calculation

```typescript
function getUpgradePath(
  from: string,
  to: string,
  available: string[]
): VersionStep[] {
  // Sort versions
  const sorted = sortVersions(available, 'asc');

  // Find versions between from and to
  const path: VersionStep[] = [];
  let current = from;

  for (const version of sorted) {
    if (compareVersions(version, current) > 0 && compareVersions(version, to) <= 0) {
      const bumpType = detectBumpType(current, version);
      const breaking = bumpType === 'MAJOR';

      path.push({
        from: current,
        to: version,
        type: bumpType,
        breaking
      });

      current = version;
    }
  }

  return path;
}
```

### Example: Multi-Step Upgrade

```typescript
const available = ["1.0.0", "1.1.0", "1.2.0", "2.0.0", "2.1.0", "3.0.0"];

// Upgrade from 1.0.0 to 3.0.0
const path = manager.getUpgradePath("1.0.0", "3.0.0", available);

console.log(path);
// [
//   { from: "1.0.0", to: "1.1.0", type: "MINOR", breaking: false },
//   { from: "1.1.0", to: "1.2.0", type: "MINOR", breaking: false },
//   { from: "1.2.0", to: "2.0.0", type: "MAJOR", breaking: true },  // ⚠️ Breaking
//   { from: "2.0.0", to: "2.1.0", type: "MINOR", breaking: false },
//   { from: "2.1.0", to: "3.0.0", type: "MAJOR", breaking: true }   // ⚠️ Breaking
// ]

// Identify breaking steps
const breakingSteps = path.filter(step => step.breaking);
console.log(`${breakingSteps.length} breaking changes in upgrade path`);
// 2 breaking changes in upgrade path
```

---

## Edge Cases

### Edge Case 1: Pre-release Versions

**Scenario**: Handling pre-release versions like `1.0.0-alpha`, `1.0.0-beta`.

**Handling**: SchemaVersionManager focuses on stable releases only. Pre-release versions should be handled separately:

```typescript
// Not supported by SchemaVersionManager
const isValid = manager.isValidVersion("1.0.0-alpha");
console.log(isValid); // false

// Recommendation: Use separate pre-release versioning system
// or strip pre-release suffix before comparison
function stripPrerelease(version: string): string {
  return version.split('-')[0];
}

console.log(stripPrerelease("1.0.0-alpha")); // "1.0.0"
```

---

### Edge Case 2: Version 0.x.x (Initial Development)

**Scenario**: Versions starting with 0 (e.g., 0.1.0, 0.2.0) during initial development.

**Handling**: Semver allows breaking changes in MINOR bumps for 0.x.x versions:

```typescript
// For 0.x.x: MINOR bumps MAY be breaking
const is0_1_breaking_to_0_2 = manager.isBreakingChange("0.1.0", "0.2.0");
// Could be true even though it's MINOR bump

// Recommendation: Treat all 0.x.x as unstable
function isStableVersion(version: string): boolean {
  const parsed = manager.parseVersion(version);
  return parsed.major >= 1;
}

console.log(isStableVersion("0.5.0")); // false (unstable)
console.log(isStableVersion("1.0.0")); // true (stable)
```

---

### Edge Case 3: Very Large Version Numbers

**Scenario**: Version numbers exceeding typical range (e.g., 100.500.9999).

**Handling**: No theoretical limit, but practical considerations:

```typescript
// Valid but unusual
const parsed = manager.parseVersion("100.500.9999");
console.log(parsed); // { major: 100, minor: 500, patch: 9999 }

// Recommendation: Set practical limits
const MAX_VERSION_COMPONENT = 10000;

function validateVersionRange(version: string): void {
  const parsed = manager.parseVersion(version);

  if (parsed.major >= MAX_VERSION_COMPONENT ||
      parsed.minor >= MAX_VERSION_COMPONENT ||
      parsed.patch >= MAX_VERSION_COMPONENT) {
    throw new Error(`Version component exceeds maximum (${MAX_VERSION_COMPONENT})`);
  }
}
```

---

### Edge Case 4: Downgrade Scenarios

**Scenario**: Determining if downgrade from v2.0.0 to v1.5.0 is safe.

**Handling**: Downgrades are generally unsafe:

```typescript
function canDowngrade(from: string, to: string): boolean {
  const comparison = manager.compareVersions(from, to);

  if (comparison <= 0) {
    // Not a downgrade
    return true;
  }

  const fromParsed = manager.parseVersion(from);
  const toParsed = manager.parseVersion(to);

  // Downgrade within same major version MAY be safe
  if (fromParsed.major === toParsed.major) {
    console.warn("Downgrade within same major version - data loss possible");
    return true;
  }

  // Downgrade across major versions is UNSAFE
  console.error("Downgrade across major versions - data loss likely");
  return false;
}

console.log(canDowngrade("1.5.0", "1.3.0")); // true (warning)
console.log(canDowngrade("2.0.0", "1.5.0")); // false (error)
```

---

### Edge Case 5: Missing Intermediate Versions

**Scenario**: Upgrade path from 1.0.0 to 3.0.0, but 2.x.x versions don't exist.

**Handling**: Direct major version jumps are allowed but risky:

```typescript
const available = ["1.0.0", "3.0.0"]; // Missing 2.x.x

const path = manager.getUpgradePath("1.0.0", "3.0.0", available);

console.log(path);
// [
//   { from: "1.0.0", to: "3.0.0", type: "MAJOR", breaking: true }
// ]

// Recommendation: Warn about skipped major versions
const skippedMajorVersions = 3 - 1 - 1; // from=1, to=3, skipped=1
console.warn(`Skipping ${skippedMajorVersions} major version(s) - review migration guide`);
```

---

### Edge Case 6: Circular Version Dependencies

**Scenario**: Schema A v2 depends on Schema B v1, but Schema B v2 depends on Schema A v2.

**Handling**: Detect and prevent circular dependencies:

```typescript
function detectCircularDependency(
  schemaId: string,
  version: string,
  dependencies: Map<string, string>,
  visited: Set<string> = new Set()
): boolean {
  const key = `${schemaId}@${version}`;

  if (visited.has(key)) {
    return true; // Circular dependency detected
  }

  visited.add(key);

  const deps = dependencies.get(key);
  if (deps) {
    for (const dep of deps) {
      if (detectCircularDependency(dep, version, dependencies, visited)) {
        return true;
      }
    }
  }

  return false;
}
```

---

### Edge Case 7: Version Comparison with Leading Zeros

**Scenario**: Versions like "01.02.03" with leading zeros.

**Handling**: Reject versions with leading zeros (not valid semver):

```typescript
function isValidVersion(version: string): boolean {
  const semverRegex = /^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)$/;
  return semverRegex.test(version);
}

console.log(isValidVersion("1.2.3"));    // true
console.log(isValidVersion("01.02.03")); // false (leading zeros)
console.log(isValidVersion("1.02.3"));   // false (leading zero in minor)
```

---

## Performance Characteristics

### Latency Targets

| Operation | Input Size | Target Latency (p95) | Notes |
|-----------|-----------|---------------------|-------|
| `parseVersion()` | 1 version | < 1ms | String parsing only |
| `compareVersions()` | 2 versions | < 1ms | Numeric comparison |
| `bumpVersion()` | 1 version | < 1ms | Simple arithmetic |
| `isBreakingChange()` | 2 schemas | < 100ms | Delegates to BackwardCompatibilityChecker |
| `calculateRequiredBump()` | 2 schemas | < 100ms | Schema analysis |
| `satisfiesRange()` | 1 version, 1 range | < 5ms | Range parsing + comparison |
| `sortVersions()` | 100 versions | < 10ms | O(n log n) sort |
| `getUpgradePath()` | 100 versions | < 20ms | Filtering + sorting |

### Throughput

- **Version Operations**: 100,000+ ops/sec (parsing, comparison, bumping)
- **Schema Analysis**: 1,000-10,000 ops/sec (breaking change detection)

### Scalability

| Version Count | Sort Performance | Notes |
|--------------|------------------|-------|
| < 100 | < 1ms | Instant |
| 100 - 1K | < 10ms | Fast |
| 1K - 10K | < 100ms | Acceptable |
| > 10K | < 1s | Consider indexing |

### Optimization Tips

**1. Cache Version Parsing**

```typescript
class CachedSchemaVersionManager {
  private parseCache = new Map<string, ParsedVersion>();

  parseVersion(version: string): ParsedVersion {
    if (this.parseCache.has(version)) {
      return this.parseCache.get(version)!;
    }

    const parsed = this.doParse(version);
    this.parseCache.set(version, parsed);
    return parsed;
  }
}
```

**2. Pre-sort Version Lists**

```typescript
// Bad: Sort on every query
function getLatest(versions: string[]): string {
  const sorted = manager.sortVersions(versions, 'desc');
  return sorted[0];
}

// Good: Sort once, cache result
class VersionCache {
  private sortedVersions: string[] | null = null;

  getLatest(versions: string[]): string {
    if (!this.sortedVersions) {
      this.sortedVersions = manager.sortVersions(versions, 'desc');
    }
    return this.sortedVersions[0];
  }
}
```

---

## Implementation Notes

### JavaScript/TypeScript Implementation

```typescript
export class SchemaVersionManager {
  parseVersion(version: string): ParsedVersion {
    const semverRegex = /^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)$/;
    const match = version.match(semverRegex);

    if (!match) {
      throw new ValidationError(`Invalid semver format: ${version}`);
    }

    return {
      major: parseInt(match[1], 10),
      minor: parseInt(match[2], 10),
      patch: parseInt(match[3], 10),
      original: version
    };
  }

  compareVersions(v1: string, v2: string): number {
    const p1 = this.parseVersion(v1);
    const p2 = this.parseVersion(v2);

    if (p1.major !== p2.major) {
      return p1.major - p2.major;
    }

    if (p1.minor !== p2.minor) {
      return p1.minor - p2.minor;
    }

    return p1.patch - p2.patch;
  }

  bumpVersion(version: string, bumpType: BumpType): string {
    const parsed = this.parseVersion(version);

    switch (bumpType) {
      case 'MAJOR':
        return `${parsed.major + 1}.0.0`;
      case 'MINOR':
        return `${parsed.major}.${parsed.minor + 1}.0`;
      case 'PATCH':
        return `${parsed.major}.${parsed.minor}.${parsed.patch + 1}`;
      default:
        throw new Error(`Invalid bump type: ${bumpType}`);
    }
  }

  async isBreakingChange(oldSchema: object, newSchema: object): Promise<boolean> {
    const checker = new BackwardCompatibilityChecker();
    const report = await checker.check(oldSchema, newSchema);
    return report.breaking_changes.length > 0;
  }

  async calculateRequiredBump(oldSchema: object, newSchema: object): Promise<BumpType> {
    const checker = new BackwardCompatibilityChecker();
    const report = await checker.check(oldSchema, newSchema);

    if (report.breaking_changes.length > 0) {
      return 'MAJOR';
    }

    if (report.additive_changes.length > 0) {
      return 'MINOR';
    }

    return 'PATCH';
  }

  satisfiesRange(version: string, range: string): boolean {
    if (range.startsWith('^')) {
      // Caret: compatible with same major version
      const baseVersion = range.substring(1);
      const baseParsed = this.parseVersion(baseVersion);
      const versionParsed = this.parseVersion(version);

      return versionParsed.major === baseParsed.major &&
             this.compareVersions(version, baseVersion) >= 0;
    }

    if (range.startsWith('~')) {
      // Tilde: compatible with same minor version
      const baseVersion = range.substring(1);
      const baseParsed = this.parseVersion(baseVersion);
      const versionParsed = this.parseVersion(version);

      return versionParsed.major === baseParsed.major &&
             versionParsed.minor === baseParsed.minor &&
             this.compareVersions(version, baseVersion) >= 0;
    }

    // Exact match
    return version === range;
  }

  sortVersions(versions: string[], order: 'asc' | 'desc' = 'asc'): string[] {
    const sorted = [...versions].sort((a, b) => this.compareVersions(a, b));
    return order === 'desc' ? sorted.reverse() : sorted;
  }

  getUpgradePath(
    fromVersion: string,
    toVersion: string,
    availableVersions: string[]
  ): VersionStep[] {
    const sorted = this.sortVersions(availableVersions, 'asc');
    const path: VersionStep[] = [];
    let current = fromVersion;

    for (const version of sorted) {
      if (this.compareVersions(version, current) > 0 &&
          this.compareVersions(version, toVersion) <= 0) {
        const fromParsed = this.parseVersion(current);
        const toParsed = this.parseVersion(version);

        let type: BumpType;
        if (toParsed.major > fromParsed.major) {
          type = 'MAJOR';
        } else if (toParsed.minor > fromParsed.minor) {
          type = 'MINOR';
        } else {
          type = 'PATCH';
        }

        path.push({
          from: current,
          to: version,
          type,
          breaking: type === 'MAJOR'
        });

        current = version;
      }
    }

    return path;
  }

  isBackwardCompatible(oldVersion: string, newVersion: string): boolean {
    const oldParsed = this.parseVersion(oldVersion);
    const newParsed = this.parseVersion(newVersion);

    // Backward compatible if same major version
    return oldParsed.major === newParsed.major;
  }

  isValidVersion(version: string): boolean {
    const semverRegex = /^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)$/;
    return semverRegex.test(version);
  }

  getLatestVersion(versions: string[]): string | null {
    if (versions.length === 0) {
      return null;
    }

    const sorted = this.sortVersions(versions, 'desc');
    return sorted[0];
  }

  checkCompatibility(v1: string, v2: string): CompatibilityReport {
    const comparison = this.compareVersions(v1, v2);
    const isUpgrade = comparison < 0;
    const isBackwardCompatible = this.isBackwardCompatible(v1, v2);

    const p1 = this.parseVersion(v1);
    const p2 = this.parseVersion(v2);

    let changeType: BumpType | null = null;
    if (p2.major > p1.major) {
      changeType = 'MAJOR';
    } else if (p2.minor > p1.minor) {
      changeType = 'MINOR';
    } else if (p2.patch > p1.patch) {
      changeType = 'PATCH';
    }

    const requiresManualMigration = changeType === 'MAJOR';
    const canAutoUpgrade = isUpgrade && !requiresManualMigration;

    let recommendation: string;
    if (!isUpgrade) {
      recommendation = "No upgrade needed (same or newer version)";
    } else if (canAutoUpgrade) {
      recommendation = "Auto-upgrade recommended (backward compatible)";
    } else {
      recommendation = "Manual migration required (breaking changes)";
    }

    return {
      isCompatible: isBackwardCompatible,
      canAutoUpgrade,
      requiresManualMigration,
      changeType,
      recommendation
    };
  }
}
```

---

## Security Considerations

### 1. Version String Validation

**Prevent Injection Attacks**:
```typescript
function sanitizeVersion(version: string): string {
  // Only allow alphanumeric and dots
  const sanitized = version.replace(/[^0-9.]/g, '');

  // Validate semver format
  if (!isValidVersion(sanitized)) {
    throw new ValidationError(`Invalid version format: ${version}`);
  }

  return sanitized;
}
```

### 2. Version Range Parsing

**Prevent ReDoS (Regular Expression Denial of Service)**:
```typescript
// Bad: Complex regex vulnerable to ReDoS
const badRegex = /^([\^~])?(\d+)\.(\d+)\.(\d+)(-[a-zA-Z0-9.]+)?(\+[a-zA-Z0-9.]+)?$/;

// Good: Simple validation
function parseRange(range: string): VersionRange {
  const prefix = range[0];
  const version = prefix === '^' || prefix === '~' ? range.substring(1) : range;

  // Validate version (simple, fast)
  if (!isValidVersion(version)) {
    throw new ValidationError(`Invalid version in range: ${range}`);
  }

  return {
    type: prefix === '^' ? 'caret' : prefix === '~' ? 'tilde' : 'exact',
    baseVersion: version,
    minVersion: version,
    maxVersion: calculateMaxVersion(version, prefix)
  };
}
```

---

## Integration Patterns

### Pattern 1: Automated Version Bumping on CI/CD

```typescript
// CI/CD pipeline step: Determine next version
async function determineNextVersion(
  currentVersion: string,
  oldSchema: object,
  newSchema: object
): Promise<string> {
  const manager = new SchemaVersionManager();

  // Calculate required bump
  const bumpType = await manager.calculateRequiredBump(oldSchema, newSchema);

  console.log(`Detected ${bumpType} change`);

  // Bump version
  const nextVersion = manager.bumpVersion(currentVersion, bumpType);

  console.log(`Next version: ${nextVersion}`);

  return nextVersion;
}

// Usage in CI/CD
const currentVersion = "1.5.0";
const nextVersion = await determineNextVersion(
  currentVersion,
  oldSchemaFromMain,
  newSchemaFromPR
);

// Update package.json or schema registry
await registry.publish({
  schema_id: "user_profile",
  version: nextVersion,
  schema: newSchemaFromPR,
  description: "Auto-generated version bump",
  published_by: "ci-bot"
});
```

### Pattern 2: Version Compatibility Middleware

```typescript
class VersionCompatibilityMiddleware {
  async handle(req: Request, res: Response, next: Function): Promise<void> {
    const requestedVersion = req.headers['schema-version'] || 'latest';
    const schemaId = req.path.split('/')[1]; // e.g., /user/123 -> "user"

    const manager = new SchemaVersionManager();

    // Get latest version
    const latest = await registry.getLatest(schemaId);

    if (!latest) {
      return res.status(404).json({ error: "Schema not found" });
    }

    let targetVersion = requestedVersion === 'latest' ? latest.version : requestedVersion;

    // Check compatibility
    const compat = manager.checkCompatibility(targetVersion, latest.version);

    if (!compat.isCompatible) {
      return res.status(400).json({
        error: "Version incompatible",
        requested: targetVersion,
        latest: latest.version,
        recommendation: compat.recommendation
      });
    }

    // Set version in request context
    req.schemaVersion = targetVersion;
    next();
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
describe('SchemaVersionManager', () => {
  let manager: SchemaVersionManager;

  beforeEach(() => {
    manager = new SchemaVersionManager();
  });

  describe('parseVersion()', () => {
    it('should parse valid semver', () => {
      const parsed = manager.parseVersion('1.2.3');

      expect(parsed.major).toBe(1);
      expect(parsed.minor).toBe(2);
      expect(parsed.patch).toBe(3);
      expect(parsed.original).toBe('1.2.3');
    });

    it('should reject invalid semver', () => {
      expect(() => manager.parseVersion('1.2')).toThrow('Invalid semver format');
      expect(() => manager.parseVersion('1.2.x')).toThrow('Invalid semver format');
      expect(() => manager.parseVersion('v1.2.3')).toThrow('Invalid semver format');
    });
  });

  describe('compareVersions()', () => {
    it('should compare versions correctly', () => {
      expect(manager.compareVersions('1.0.0', '2.0.0')).toBe(-1);
      expect(manager.compareVersions('2.0.0', '1.0.0')).toBe(1);
      expect(manager.compareVersions('1.2.3', '1.2.3')).toBe(0);
    });

    it('should handle semantic comparison', () => {
      expect(manager.compareVersions('1.10.0', '1.2.0')).toBe(1); // 1.10 > 1.2
      expect(manager.compareVersions('1.2.10', '1.2.2')).toBe(1); // 1.2.10 > 1.2.2
    });
  });

  describe('bumpVersion()', () => {
    it('should bump patch version', () => {
      expect(manager.bumpVersion('1.2.3', 'PATCH')).toBe('1.2.4');
    });

    it('should bump minor version and reset patch', () => {
      expect(manager.bumpVersion('1.2.3', 'MINOR')).toBe('1.3.0');
    });

    it('should bump major version and reset minor/patch', () => {
      expect(manager.bumpVersion('1.2.3', 'MAJOR')).toBe('2.0.0');
    });
  });

  describe('satisfiesRange()', () => {
    it('should check caret range', () => {
      expect(manager.satisfiesRange('1.2.3', '^1.0.0')).toBe(true);
      expect(manager.satisfiesRange('1.5.0', '^1.0.0')).toBe(true);
      expect(manager.satisfiesRange('2.0.0', '^1.0.0')).toBe(false);
    });

    it('should check tilde range', () => {
      expect(manager.satisfiesRange('1.2.3', '~1.2.0')).toBe(true);
      expect(manager.satisfiesRange('1.2.9', '~1.2.0')).toBe(true);
      expect(manager.satisfiesRange('1.3.0', '~1.2.0')).toBe(false);
    });

    it('should check exact match', () => {
      expect(manager.satisfiesRange('1.2.3', '1.2.3')).toBe(true);
      expect(manager.satisfiesRange('1.2.4', '1.2.3')).toBe(false);
    });
  });

  describe('sortVersions()', () => {
    it('should sort versions ascending', () => {
      const versions = ['1.10.0', '1.2.0', '2.0.0', '1.2.3'];
      const sorted = manager.sortVersions(versions, 'asc');

      expect(sorted).toEqual(['1.2.0', '1.2.3', '1.10.0', '2.0.0']);
    });

    it('should sort versions descending', () => {
      const versions = ['1.10.0', '1.2.0', '2.0.0', '1.2.3'];
      const sorted = manager.sortVersions(versions, 'desc');

      expect(sorted).toEqual(['2.0.0', '1.10.0', '1.2.3', '1.2.0']);
    });
  });

  describe('isBackwardCompatible()', () => {
    it('should return true for same major version', () => {
      expect(manager.isBackwardCompatible('1.0.0', '1.5.0')).toBe(true);
    });

    it('should return false for different major version', () => {
      expect(manager.isBackwardCompatible('1.5.0', '2.0.0')).toBe(false);
    });
  });
});
```

---

## Migration Guide

### From Manual Versioning to SchemaVersionManager

```typescript
// Before: Manual version assignment
const nextVersion = "1.3.0"; // Guessed by developer

await registry.publish({
  schema_id: "user_profile",
  version: nextVersion,
  schema: newSchema,
  description: "Added email field"
});

// After: Automated version bumping
const manager = new SchemaVersionManager();

const currentVersion = "1.2.0";
const bumpType = await manager.calculateRequiredBump(oldSchema, newSchema);
const nextVersion = manager.bumpVersion(currentVersion, bumpType);

console.log(`Auto-detected ${bumpType} bump: ${currentVersion} → ${nextVersion}`);

await registry.publish({
  schema_id: "user_profile",
  version: nextVersion,
  schema: newSchema,
  description: "Added email field"
});
```

---

## Related Primitives

- **SchemaRegistry**: Centralized schema storage using versions from SchemaVersionManager
- **BackwardCompatibilityChecker**: Detects breaking changes used by calculateRequiredBump()
- **MigrationEngine**: Uses version comparison for migration planning
- **ParserVersionManager**: Similar versioning logic for parser versions

---

## References

- [Semantic Versioning 2.0.0](https://semver.org/)
- [npm semver package](https://github.com/npm/node-semver)
- [Objective Layer Architecture](../architecture/objective-layer.md)
- [Schema Registry Vertical 5.2](../verticals/schema-registry.md)

---

**End of SchemaVersionManager OL Primitive Specification**
