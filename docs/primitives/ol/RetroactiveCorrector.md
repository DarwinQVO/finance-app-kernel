# RetroactiveCorrector OL Primitive

**Domain:** Provenance Ledger (Vertical 5.1)
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
8. [Validation Rules](#validation-rules)
9. [Impact Analysis](#impact-analysis)
10. [Correction Workflows](#correction-workflows)
11. [Edge Cases](#edge-cases)
12. [Performance Characteristics](#performance-characteristics)
13. [Implementation Notes](#implementation-notes)
14. [Audit & Compliance](#audit--compliance)
15. [Integration Patterns](#integration-patterns)
16. [Multi-Domain Examples](#multi-domain-examples)
17. [Testing Strategy](#testing-strategy)
18. [Migration Guide](#migration-guide)
19. [Related Primitives](#related-primitives)
20. [References](#references)

---

## Overview

The **RetroactiveCorrector** primitive handles retroactive corrections to historical data with comprehensive validation, impact analysis, and audit trail integration. It ensures retroactive changes are properly validated, documented, and their downstream effects are understood.

### Key Capabilities

- **Retroactive Correction**: Apply corrections to historical data with proper valid_time
- **Validation**: Ensure corrections are logically valid and authorized
- **Impact Analysis**: Identify downstream entities affected by correction
- **Automatic Provenance**: Append correction event to provenance ledger
- **Workflow Support**: Multi-step approval workflow for sensitive corrections
- **Audit Trail**: Complete trail of who corrected what, when, and why

### Design Philosophy

The RetroactiveCorrector follows four core principles:

1. **Safety First**: Validate all corrections before applying
2. **Transparency**: Full audit trail for all retroactive changes
3. **Impact Awareness**: Analyze downstream effects before committing
4. **Compliance**: Support regulatory requirements for data corrections

---

## Purpose & Scope

### Problem Statement

Retroactive corrections are necessary but risky:

**Why Retroactive Corrections Are Needed**:
- Data entry errors discovered later
- Late-arriving information (e.g., bank corrections, delayed diagnoses)
- Accounting adjustments (e.g., year-end corrections)
- Compliance corrections (e.g., tax amendments)

**Risks Without Proper Handling**:
- L Inconsistent state across related entities
- L No audit trail of who changed what
- L Unknown downstream impact
- L Regulatory compliance violations
- L Lost trust in data integrity

**RetroactiveCorrector Solution**:
-  Validates correction is logically sound
-  Analyzes impact on downstream entities
-  Creates complete audit trail
-  Supports approval workflows
-  Maintains data integrity

### Solution

RetroactiveCorrector implements comprehensive retroactive correction handling with:

1. **Validation Engine**: Ensure corrections are valid and authorized
2. **Impact Analyzer**: Identify affected downstream entities
3. **Provenance Integration**: Automatic event logging
4. **Workflow Engine**: Multi-step approval for sensitive corrections
5. **Rollback Support**: Ability to undo corrections if needed

---

## Multi-Domain Applicability

RetroactiveCorrector is critical across domains requiring data accuracy. 7+ domains:

### 1. Finance

**Use Case**: Correct transaction amounts, categories, or dates discovered to be wrong.

**Corrections**:
- Bank statement errors
- Accounting period adjustments
- Tax year corrections
- Revenue recognition adjustments

**Compliance**: SOX requires audit trail for all financial corrections.

**Example**:
```typescript
// Correct transaction amount discovered to be wrong
const correction = await retroactiveCorrector.correct({
  entity_id: "txn_001",
  field_name: "amount",
  old_value: 100.00,
  new_value: 105.00,
  effective_date: "2024-12-15T10:00:00Z", // Original transaction date
  reason: "Corrected from receipt - original extraction was $100 but receipt shows $105",
  user_id: "accountant_jane_doe",
  requires_approval: true // Sensitive correction
});

// Impact analysis shows affected reports
console.log(`Impact: ${correction.impact_analysis.affected_entities.length} entities affected`);
// - December revenue report
// - Q4 financial summary
// - Annual tax calculations
```

### 2. Healthcare

**Use Case**: Update diagnosis codes based on new test results.

**Corrections**:
- Diagnosis code updates
- Lab result corrections
- Treatment plan retroactive changes
- Insurance claim corrections

**Compliance**: HIPAA requires tracking all patient record corrections.

**Example**:
```typescript
// Update diagnosis retroactively after lab results
const correction = await retroactiveCorrector.correct({
  entity_id: "encounter_12345",
  field_name: "diagnosis_code",
  old_value: "R07.9", // Chest pain, unspecified
  new_value: "I20.9", // Angina pectoris
  effective_date: "2025-01-15T14:30:00Z", // Original encounter date
  reason: "Updated diagnosis based on EKG results received 2025-01-18",
  user_id: "dr_jones",
  supporting_evidence: ["ekg_report_12345.pdf"]
});

// Impact analysis
console.log(`Affected claims: ${correction.impact_analysis.insurance_claims.length}`);
// Insurance claim needs to be resubmitted with updated diagnosis code
```

### 3. Legal

**Use Case**: Correct case filing dates or evidence collection timestamps.

**Corrections**:
- Evidence collection date corrections
- Document filing timestamp corrections
- Case status retroactive changes

**Compliance**: Legal discovery requires proving when corrections were made.

**Example**:
```typescript
// Correct evidence collection date
const correction = await retroactiveCorrector.correct({
  entity_id: "evidence_456",
  field_name: "collection_date",
  old_value: "2024-12-15T09:00:00Z",
  new_value: "2024-12-14T15:30:00Z",
  effective_date: "2024-12-14T15:30:00Z",
  reason: "Corrected based on chain of custody log review",
  user_id: "legal_admin_sarah",
  requires_approval: true,
  approver_id: "attorney_michael_brown"
});
```

### 4. Research (RSRCH - Utilitario)

**Use Case**: Enrich facts retroactively as new sources are discovered.

**Context**: RSRCH facts are initially vague (extracted from tweets/headlines) and get enriched with specific details from interviews/podcasts. Corrections are retroactive to the original fact date.

**Corrections**:
- Fact enrichment (vague → specific amount, date, details)
- Entity normalization (@sama → Sam Altman)
- Multi-source confirmation (add corroborating source)
- Contradiction resolution (conflicting values from different sources)
- Fact merger (duplicate facts from different sources unified)

**Compliance**: Provenance transparency - track which source contributed which field, when discovered, effective retroactively.

**Example (Enrich Investment Fact)**:
```typescript
// Scenario: Initial TechCrunch article (Jan 15) said "Sam Altman invested in Helion"
// Podcast discovered (Feb 20) reveals the amount was $375 million
// Retroactively enrich the fact effective Jan 15 (when investment actually happened)

const enrichment = await retroactiveCorrector.correct({
  entity_id: "fact_sama_helion_001",
  field_name: "investment_amount",
  old_value: null, // Initially unknown
  new_value: 375000000, // $375 million
  effective_date: "2025-01-15T00:00:00Z", // Investment date (retroactive to Jan 15)
  reason: "Amount revealed in Lex Fridman podcast episode #456 with Sam Altman",
  user_id: "podcast_parser_lex_fridman",
  metadata: {
    source_url: "https://youtube.com/watch?v=xyz",
    source_type: "podcast_transcript",
    source_credibility: 0.95,
    quote: "I invested three hundred and seventy-five million dollars in Helion",
    timestamp_in_video: "1:23:45"
  }
});

console.log(`Fact enriched: ${enrichment.field_name}`);
console.log(`  Old value: ${enrichment.old_value} (unknown)`);
console.log(`  New value: $${enrichment.new_value / 1000000}M`);
console.log(`  Effective: ${enrichment.effective_date} (retroactive ${enrichment.retroactive_days} days)`);
console.log(`  Source: Lex Fridman podcast`);
console.log(`  Confidence: 0.95 (first-person interview)`);
```

**RSRCH-Specific Patterns**:
- **Retroactive Enrichment**: Vague fact from tweet → Specific fact from podcast (retroactive to tweet date)
- **Multi-Source Confirmation**: TechCrunch says "$300-400M" → Podcast confirms "$375M" → Retroactively update to exact amount
- **Entity Normalization**: @sama (tweet) → sama (informal) → Sam Altman (canonical) - all retroactive to original mention date

### 5. E-commerce

**Use Case**: Correct product prices or inventory levels.

**Corrections**:
- Price error corrections
- Inventory count adjustments
- Product attribute corrections

**Compliance**: Consumer protection requires price correction audit trail.

**Example**:
```typescript
// Correct product price
const correction = await retroactiveCorrector.correct({
  entity_id: "prod_001",
  field_name: "price",
  old_value: 29.99,
  new_value: 27.99,
  effective_date: "2025-01-10T00:00:00Z",
  reason: "Price was incorrectly set - should have been $27.99 during sale",
  user_id: "pricing_manager_tom"
});

// Impact analysis shows affected orders
const affectedOrders = correction.impact_analysis.orders_needing_refund;
console.log(`${affectedOrders.length} customers overpaid - refund needed`);
```

### 6. SaaS

**Use Case**: Correct subscription plan or billing data.

**Corrections**:
- Plan start date corrections
- Billing amount corrections
- Feature entitlement corrections

**Compliance**: Revenue recognition (ASC 606) requires correction documentation.

**Example**:
```typescript
// Correct subscription start date
const correction = await retroactiveCorrector.correct({
  entity_id: "sub_001",
  field_name: "start_date",
  old_value: "2025-01-01T00:00:00Z",
  new_value: "2024-12-15T00:00:00Z",
  effective_date: "2024-12-15T00:00:00Z",
  reason: "Customer actually started trial on Dec 15, not Jan 1",
  user_id: "billing_admin_lisa"
});

// Impact analysis
console.log(`Billing adjustments needed: ${correction.impact_analysis.invoices_to_adjust.length}`);
// Pro-rated billing adjustment required
```

### 7. Insurance

**Use Case**: Correct policy effective dates or premium amounts.

**Corrections**:
- Policy effective date corrections
- Premium amount corrections
- Coverage start/end date corrections

**Compliance**: Insurance regulations require audit trail for policy corrections.

**Example**:
```typescript
// Correct policy effective date
const correction = await retroactiveCorrector.correct({
  entity_id: "policy_789",
  field_name: "effective_date",
  old_value: "2025-01-01T00:00:00Z",
  new_value: "2024-12-20T00:00:00Z",
  effective_date: "2024-12-20T00:00:00Z",
  reason: "Coverage actually started Dec 20 per signed application",
  user_id: "underwriter_mike",
  requires_approval: true
});

// Impact analysis
console.log(`Claims affected: ${correction.impact_analysis.claims_to_reprocess.length}`);
```

### Cross-Domain Benefits

All domains benefit from:
- **Data Quality**: Ability to fix errors retroactively
- **Compliance**: Complete audit trail for corrections
- **Transparency**: Clear documentation of who changed what and why
- **Impact Awareness**: Understanding downstream effects before committing

---

## Core Concepts

### Retroactive Correction

A retroactive correction is a change to historical data where:
- `valid_time` (when it was effective) is in the past
- `transaction_time` (when we're making the correction) is now
- The correction applies retroactively to the original effective date

**Example**:
```
Original Event:
  transaction_time: 2025-01-15T10:00:00Z
  valid_time: 2025-01-15T10:00:00Z
  amount: $100.00

Retroactive Correction:
  transaction_time: 2025-01-20T14:30:00Z (now)
  valid_time: 2025-01-15T10:00:00Z (retroactive to original date)
  amount: $105.00

Result: We now know the amount was always $105.00, but we only learned this on Jan 20.
```

### Validation Rules

Before applying a correction, validate:

1. **Temporal Validity**: `effective_date` must be in the past or present
2. **Value Change**: `new_value` must differ from `old_value`
3. **Authorization**: User must have permission to correct this field
4. **Approval**: Sensitive corrections require approval
5. **Impact Assessment**: Downstream impact must be acceptable

### Impact Types

Corrections can impact:

1. **Direct Impact**: The entity being corrected
2. **Dependent Impact**: Entities that reference this entity
3. **Calculated Impact**: Aggregations/reports that include this entity
4. **Workflow Impact**: Processes triggered by the original value

---

## Interface Definition

### TypeScript Interface

```typescript
interface RetroactiveCorrector {
  /**
   * Apply retroactive correction with validation
   *
   * @param correction - Correction details
   * @returns CorrectionResult with impact analysis
   * @throws ValidationError if correction is invalid
   */
  correct(
    correction: RetroactiveCorrection
  ): Promise<CorrectionResult>;

  /**
   * Validate correction without applying it
   *
   * @param correction - Correction to validate
   * @returns Validation result with errors/warnings
   */
  validateRetroactiveChange(
    correction: RetroactiveCorrection
  ): Promise<ValidationResult>;

  /**
   * Get impact analysis for proposed correction
   *
   * @param correction - Proposed correction
   * @returns Impact analysis showing affected entities
   */
  getImpactAnalysis(
    correction: RetroactiveCorrection
  ): Promise<ImpactAnalysis>;

  /**
   * Request approval for sensitive correction
   *
   * @param correction - Correction requiring approval
   * @returns Approval request ID
   */
  requestApproval(
    correction: RetroactiveCorrection
  ): Promise<string>;

  /**
   * Approve pending correction
   *
   * @param approvalId - Approval request ID
   * @param approverId - User approving
   * @returns Applied correction result
   */
  approve(
    approvalId: string,
    approverId: string
  ): Promise<CorrectionResult>;

  /**
   * Reject pending correction
   *
   * @param approvalId - Approval request ID
   * @param approverId - User rejecting
   * @param reason - Why rejected
   */
  reject(
    approvalId: string,
    approverId: string,
    reason: string
  ): Promise<void>;

  /**
   * Rollback a correction (create reverse correction)
   *
   * @param correctionId - Original correction to rollback
   * @param reason - Why rolling back
   * @returns Rollback correction result
   */
  rollback(
    correctionId: string,
    reason: string
  ): Promise<CorrectionResult>;

  /**
   * Get correction history for entity/field
   *
   * @param entityId - Entity to query
   * @param fieldName - Optional field filter
   * @returns All corrections for entity
   */
  getCorrectionHistory(
    entityId: string,
    fieldName?: string
  ): Promise<CorrectionRecord[]>;
}
```

---

## Data Model

### RetroactiveCorrection Type

```typescript
interface RetroactiveCorrection {
  // Entity Being Corrected
  entity_id: string;
  entity_type?: string;

  // Field Being Corrected
  field_name: string;

  // Values
  old_value: any;                  // Current (incorrect) value
  new_value: any;                  // Corrected value

  // Temporal
  effective_date: string;          // When correction is effective (valid_time)

  // Metadata
  reason: string;                  // Why correction is needed (required)
  user_id: string;                 // Who is making correction

  // Approval
  requires_approval?: boolean;     // Default: false
  approver_id?: string;            // Required if requires_approval

  // Supporting Evidence
  supporting_evidence?: string[];  // URLs/references to documents
}
```

### CorrectionResult Type

```typescript
interface CorrectionResult {
  // Correction Details
  correction_id: string;
  entity_id: string;
  field_name: string;
  old_value: any;
  new_value: any;

  // Status
  status: 'applied' | 'pending_approval' | 'rejected';
  applied_at?: string;             // When correction was applied

  // Provenance
  provenance_record_id: string;    // Reference to provenance ledger event

  // Impact Analysis
  impact_analysis: ImpactAnalysis;

  // Approval
  approval_request_id?: string;
  approver_id?: string;
  approved_at?: string;
}
```

### ValidationResult Type

```typescript
interface ValidationResult {
  is_valid: boolean;

  // Errors (must fix before applying)
  errors: ValidationError[];

  // Warnings (can apply but should review)
  warnings: ValidationWarning[];
}

interface ValidationError {
  code: string;
  message: string;
  field?: string;
}

interface ValidationWarning {
  code: string;
  message: string;
  severity: 'low' | 'medium' | 'high';
}
```

### ImpactAnalysis Type

```typescript
interface ImpactAnalysis {
  // Direct Impact
  entity_id: string;
  field_name: string;
  value_change: { from: any; to: any };

  // Dependent Entities
  affected_entities: AffectedEntity[];

  // Calculated Fields/Reports
  recalculation_needed: RecalculationNeeded[];

  // Workflows Triggered
  workflows_to_rerun: WorkflowReference[];

  // Estimated Impact
  impact_severity: 'low' | 'medium' | 'high' | 'critical';
  estimated_affected_count: number;
}

interface AffectedEntity {
  entity_id: string;
  entity_type: string;
  relationship: string;           // How it's related (e.g., "child", "reference")
  impact_type: string;            // e.g., "needs_recalculation", "needs_review"
}

interface RecalculationNeeded {
  calculation_type: string;       // e.g., "revenue_report", "tax_summary"
  calculation_id: string;
  reason: string;
}

interface WorkflowReference {
  workflow_id: string;
  workflow_type: string;
  reason: string;
}
```

### CorrectionRecord Type

```typescript
interface CorrectionRecord {
  correction_id: string;
  entity_id: string;
  field_name: string;
  old_value: any;
  new_value: any;
  effective_date: string;
  applied_at: string;
  user_id: string;
  reason: string;
  status: string;
  provenance_record_id: string;
}
```

---

## Core Functionality

### 1. correct()

Apply retroactive correction with validation and impact analysis.

#### Signature

```typescript
correct(
  correction: RetroactiveCorrection
): Promise<CorrectionResult>
```

#### Behavior

1. **Validate Correction**:
   - Call `validateRetroactiveChange()`
   - If validation fails, throw error

2. **Check Approval Requirement**:
   - If `requires_approval === true`, create approval request
   - Return with status `pending_approval`

3. **Impact Analysis**:
   - Call `getImpactAnalysis()`
   - Include in result

4. **Apply Correction**:
   - Append event to provenance ledger with:
     - `valid_time = correction.effective_date`
     - `transaction_time = NOW()`
     - `event_type = 'corrected'`

5. **Return Result**:
   - Status: `applied`
   - Include provenance record ID
   - Include impact analysis

#### Example

```typescript
// Apply correction
const result = await retroactiveCorrector.correct({
  entity_id: "txn_001",
  field_name: "amount",
  old_value: 100.00,
  new_value: 105.00,
  effective_date: "2025-01-15T10:00:00Z",
  reason: "Corrected from receipt",
  user_id: "accountant_jane"
});

console.log(`Correction ${result.correction_id} applied`);
console.log(`Provenance record: ${result.provenance_record_id}`);
console.log(`Impact: ${result.impact_analysis.affected_entities.length} entities affected`);
```

#### Implementation

```typescript
async correct(
  correction: RetroactiveCorrection
): Promise<CorrectionResult> {
  // 1. Validate
  const validation = await this.validateRetroactiveChange(correction);

  if (!validation.is_valid) {
    throw new ValidationError(
      `Validation failed: ${validation.errors.map(e => e.message).join(', ')}`
    );
  }

  // 2. Check approval requirement
  if (correction.requires_approval) {
    const approvalId = await this.requestApproval(correction);
    return {
      correction_id: generateId(),
      entity_id: correction.entity_id,
      field_name: correction.field_name,
      old_value: correction.old_value,
      new_value: correction.new_value,
      status: 'pending_approval',
      approval_request_id: approvalId,
      provenance_record_id: '',
      impact_analysis: await this.getImpactAnalysis(correction)
    };
  }

  // 3. Get impact analysis
  const impact = await this.getImpactAnalysis(correction);

  // 4. Append to provenance ledger
  await provenanceLedger.append({
    entity_id: correction.entity_id,
    entity_type: correction.entity_type || 'unknown',
    event_type: 'corrected',
    field_name: correction.field_name,
    old_value: correction.old_value,
    new_value: correction.new_value,
    valid_time: correction.effective_date,
    transaction_time: new Date().toISOString(),
    user_id: correction.user_id,
    reason: correction.reason
  });

  // 5. Get provenance record ID
  const history = await provenanceLedger.getHistory(correction.entity_id, correction.field_name);
  const provenanceRecordId = history[history.length - 1].record_id;

  // 6. Return result
  return {
    correction_id: generateId(),
    entity_id: correction.entity_id,
    field_name: correction.field_name,
    old_value: correction.old_value,
    new_value: correction.new_value,
    status: 'applied',
    applied_at: new Date().toISOString(),
    provenance_record_id: provenanceRecordId,
    impact_analysis: impact
  };
}
```

---

### 2. validateRetroactiveChange()

Validate correction without applying it.

#### Signature

```typescript
validateRetroactiveChange(
  correction: RetroactiveCorrection
): Promise<ValidationResult>
```

#### Behavior

Run validation checks:

1. **Required Fields**: All required fields present
2. **Temporal Validity**: `effective_date` is valid timestamp
3. **Value Change**: `new_value !== old_value`
4. **Authorization**: User has permission to correct this field
5. **Effective Date**: Not in far future (warn if > 1 year future)

#### Example

```typescript
// Validate before applying
const validation = await retroactiveCorrector.validateRetroactiveChange({
  entity_id: "txn_001",
  field_name: "amount",
  old_value: 100.00,
  new_value: 105.00,
  effective_date: "2025-01-15T10:00:00Z",
  reason: "Corrected from receipt",
  user_id: "accountant_jane"
});

if (!validation.is_valid) {
  console.error("Validation failed:");
  for (const error of validation.errors) {
    console.error(`- ${error.message}`);
  }
} else if (validation.warnings.length > 0) {
  console.warn("Warnings:");
  for (const warning of validation.warnings) {
    console.warn(`- ${warning.message}`);
  }
}
```

#### Implementation

```typescript
async validateRetroactiveChange(
  correction: RetroactiveCorrection
): Promise<ValidationResult> {
  const errors: ValidationError[] = [];
  const warnings: ValidationWarning[] = [];

  // 1. Required fields
  if (!correction.entity_id) {
    errors.push({
      code: 'MISSING_ENTITY_ID',
      message: 'entity_id is required',
      field: 'entity_id'
    });
  }

  if (!correction.field_name) {
    errors.push({
      code: 'MISSING_FIELD_NAME',
      message: 'field_name is required',
      field: 'field_name'
    });
  }

  if (!correction.reason) {
    errors.push({
      code: 'MISSING_REASON',
      message: 'reason is required for all corrections',
      field: 'reason'
    });
  }

  // 2. Temporal validity
  if (!correction.effective_date) {
    errors.push({
      code: 'MISSING_EFFECTIVE_DATE',
      message: 'effective_date is required',
      field: 'effective_date'
    });
  } else {
    const effectiveDate = new Date(correction.effective_date);
    if (isNaN(effectiveDate.getTime())) {
      errors.push({
        code: 'INVALID_EFFECTIVE_DATE',
        message: 'effective_date must be valid ISO timestamp',
        field: 'effective_date'
      });
    }

    // Warn if far future
    const oneYearFromNow = new Date();
    oneYearFromNow.setFullYear(oneYearFromNow.getFullYear() + 1);
    if (effectiveDate > oneYearFromNow) {
      warnings.push({
        code: 'FUTURE_EFFECTIVE_DATE',
        message: 'effective_date is more than 1 year in the future',
        severity: 'medium'
      });
    }
  }

  // 3. Value change
  if (correction.old_value === correction.new_value) {
    errors.push({
      code: 'NO_VALUE_CHANGE',
      message: 'new_value must differ from old_value',
      field: 'new_value'
    });
  }

  // 4. Authorization
  const hasPermission = await this.checkPermission(
    correction.user_id,
    correction.entity_type,
    correction.field_name
  );

  if (!hasPermission) {
    errors.push({
      code: 'UNAUTHORIZED',
      message: `User ${correction.user_id} not authorized to correct ${correction.field_name}`,
      field: 'user_id'
    });
  }

  // 5. Approval requirement
  if (correction.requires_approval && !correction.approver_id) {
    errors.push({
      code: 'MISSING_APPROVER',
      message: 'approver_id required when requires_approval is true',
      field: 'approver_id'
    });
  }

  return {
    is_valid: errors.length === 0,
    errors,
    warnings
  };
}
```

---

### 3. getImpactAnalysis()

Analyze downstream impact of proposed correction.

#### Signature

```typescript
getImpactAnalysis(
  correction: RetroactiveCorrection
): Promise<ImpactAnalysis>
```

#### Behavior

1. **Identify Affected Entities**:
   - Query entities that reference this entity
   - Find calculated fields that use this value

2. **Assess Impact Severity**:
   - Low: Few entities affected, non-critical field
   - Medium: Moderate entities affected, important field
   - High: Many entities affected, critical field
   - Critical: System-wide impact, financial/compliance impact

3. **Build Impact Report**:
   - List of affected entities
   - Calculations needing re-run
   - Workflows to trigger

#### Example

```typescript
// Analyze impact before applying
const impact = await retroactiveCorrector.getImpactAnalysis({
  entity_id: "txn_001",
  field_name: "amount",
  old_value: 100.00,
  new_value: 105.00,
  effective_date: "2024-12-15T10:00:00Z",
  reason: "Correction",
  user_id: "accountant_jane"
});

console.log(`Impact Severity: ${impact.impact_severity}`);
console.log(`Affected Entities: ${impact.affected_entities.length}`);

for (const entity of impact.affected_entities) {
  console.log(`- ${entity.entity_type} ${entity.entity_id}: ${entity.impact_type}`);
}

console.log(`Recalculations Needed: ${impact.recalculation_needed.length}`);
for (const calc of impact.recalculation_needed) {
  console.log(`- ${calc.calculation_type}: ${calc.reason}`);
}
```

---

### 4. requestApproval()

Request approval for sensitive correction.

#### Signature

```typescript
requestApproval(
  correction: RetroactiveCorrection
): Promise<string>
```

#### Behavior

1. Create approval request record
2. Notify approver
3. Return approval request ID

#### Example

```typescript
const approvalId = await retroactiveCorrector.requestApproval({
  entity_id: "txn_001",
  field_name: "amount",
  old_value: 100.00,
  new_value: 105.00,
  effective_date: "2024-12-15T10:00:00Z",
  reason: "Correction from receipt",
  user_id: "accountant_jane",
  requires_approval: true,
  approver_id: "manager_bob"
});

console.log(`Approval request ${approvalId} created`);
console.log("Waiting for manager_bob to approve...");
```

---

### 5. approve()

Approve pending correction.

#### Signature

```typescript
approve(
  approvalId: string,
  approverId: string
): Promise<CorrectionResult>
```

#### Behavior

1. Fetch approval request
2. Verify approverId matches requested approver
3. Apply correction
4. Update approval status
5. Return correction result

#### Example

```typescript
// Approver approves the correction
const result = await retroactiveCorrector.approve(
  "approval_12345",
  "manager_bob"
);

console.log(`Correction approved and applied`);
console.log(`Provenance record: ${result.provenance_record_id}`);
```

---

### 6. rollback()

Rollback a correction by creating reverse correction.

#### Signature

```typescript
rollback(
  correctionId: string,
  reason: string
): Promise<CorrectionResult>
```

#### Behavior

1. Fetch original correction
2. Create reverse correction:
   - `old_value` = original `new_value`
   - `new_value` = original `old_value`
   - `effective_date` = original `effective_date`
3. Apply reverse correction
4. Return result

#### Example

```typescript
// Rollback a correction
const rollbackResult = await retroactiveCorrector.rollback(
  "correction_12345",
  "Correction was incorrect - reverting to original value"
);

console.log(`Correction rolled back`);
console.log(`Rollback provenance: ${rollbackResult.provenance_record_id}`);
```

---

## Validation Rules

### Rule 1: Temporal Consistency

**Rule**: `effective_date` must be a valid timestamp.

**Validation**:
```typescript
const effectiveDate = new Date(correction.effective_date);
if (isNaN(effectiveDate.getTime())) {
  errors.push({
    code: 'INVALID_TIMESTAMP',
    message: 'effective_date must be valid ISO timestamp'
  });
}
```

---

### Rule 2: Value Change Required

**Rule**: `new_value` must differ from `old_value`.

**Validation**:
```typescript
if (JSON.stringify(correction.old_value) === JSON.stringify(correction.new_value)) {
  errors.push({
    code: 'NO_CHANGE',
    message: 'new_value must differ from old_value'
  });
}
```

---

### Rule 3: Reason Required

**Rule**: All corrections must have a documented reason.

**Validation**:
```typescript
if (!correction.reason || correction.reason.trim().length === 0) {
  errors.push({
    code: 'MISSING_REASON',
    message: 'reason is required and cannot be empty'
  });
}
```

---

### Rule 4: Authorization

**Rule**: User must have permission to correct this field.

**Validation**:
```typescript
const hasPermission = await authorizationService.canCorrect(
  correction.user_id,
  correction.entity_type,
  correction.field_name
);

if (!hasPermission) {
  errors.push({
    code: 'UNAUTHORIZED',
    message: 'User not authorized to correct this field'
  });
}
```

---

### Rule 5: Approval for Sensitive Fields

**Rule**: Financial fields require manager approval.

**Validation**:
```typescript
const sensitiveFields = ['amount', 'revenue', 'cost', 'premium'];

if (sensitiveFields.includes(correction.field_name)) {
  if (!correction.requires_approval) {
    warnings.push({
      code: 'APPROVAL_RECOMMENDED',
      message: 'Approval recommended for sensitive financial field',
      severity: 'high'
    });
  }
}
```

---

## Impact Analysis

### Impact Severity Levels

**Low**:
- Single entity affected
- Non-critical field
- No downstream calculations

**Medium**:
- Multiple entities affected (< 10)
- Important field
- Some calculations need re-run

**High**:
- Many entities affected (10-100)
- Critical field
- Many calculations need re-run

**Critical**:
- System-wide impact (> 100 entities)
- Financial/compliance field
- Reports need regeneration

### Impact Analysis Algorithm

```
ALGORITHM: AnalyzeImpact(correction)

1. Find Direct Dependencies:
   - Query entities that reference this entity
   - Find calculated fields using this value

2. Find Indirect Dependencies:
   - For each direct dependency:
     - Recursively find its dependencies
   - Build dependency graph

3. Assess Calculations:
   - Find reports that include this entity
   - Find aggregations that sum/average this field

4. Estimate Severity:
   - Count total affected entities
   - Check if field is financial/compliance-critical
   - Apply severity rules

5. Return Impact Analysis:
   - List of affected entities
   - Calculations needing re-run
   - Severity assessment
```

---

## Correction Workflows

### Workflow 1: Simple Correction

1. User initiates correction
2. System validates
3. System applies correction
4. Provenance event logged

### Workflow 2: Approval-Required Correction

1. User initiates correction
2. System validates
3. System creates approval request
4. Approver reviews impact analysis
5. Approver approves/rejects
6. If approved, system applies correction
7. Provenance event logged

### Workflow 3: Batch Correction

1. User uploads CSV of corrections
2. System validates all corrections
3. System shows aggregate impact analysis
4. User confirms batch
5. System applies all corrections
6. Provenance events logged for each

---

## Edge Cases

### Edge Case 1: Correction of Already-Corrected Field

**Scenario**: Field was already corrected once, now correcting again.

**Handling**: Allow multiple corrections, chain in provenance ledger:

```typescript
// First correction
await retroactiveCorrector.correct({
  entity_id: "txn_001",
  field_name: "amount",
  old_value: 100.00,
  new_value: 105.00,
  effective_date: "2025-01-15T10:00:00Z",
  reason: "First correction from receipt",
  user_id: "user_jane"
});

// Second correction
await retroactiveCorrector.correct({
  entity_id: "txn_001",
  field_name: "amount",
  old_value: 105.00,
  new_value: 107.00,
  effective_date: "2025-01-15T10:00:00Z",
  reason: "Second correction - found another receipt",
  user_id: "user_jane"
});

// Timeline shows both corrections
```

---

### Edge Case 2: Conflicting Corrections

**Scenario**: Two users correct same field simultaneously.

**Handling**: Last correction wins, both in provenance ledger:

```typescript
// User A corrects to 105
// User B corrects to 107 (simultaneously)

// Both corrections recorded in provenance ledger
// Final value: 107 (whichever was last by transaction_time)
```

---

### Edge Case 3: Correction Exceeds Threshold

**Scenario**: Correction amount exceeds allowed threshold.

**Handling**: Warn user, require approval:

```typescript
const validation = await retroactiveCorrector.validateRetroactiveChange({
  entity_id: "txn_001",
  field_name: "amount",
  old_value: 100.00,
  new_value: 500.00, // 400% increase
  effective_date: "2025-01-15T10:00:00Z",
  reason: "Correction",
  user_id: "user_jane"
});

// Warning: Large correction amount
validation.warnings.push({
  code: 'LARGE_CORRECTION',
  message: 'Correction exceeds 200% threshold - approval recommended',
  severity: 'high'
});
```

---

### Edge Case 4: Circular Dependency

**Scenario**: Entity A depends on B, B depends on A.

**Handling**: Detect cycle in impact analysis:

```typescript
const impact = await retroactiveCorrector.getImpactAnalysis({
  entity_id: "entity_A",
  field_name: "amount",
  ...
});

// Circular dependency detected
if (impact.circular_dependency_detected) {
  console.warn("Circular dependency detected - manual review needed");
}
```

---

### Edge Case 5: Correction of Deleted Entity

**Scenario**: Attempting to correct field of deleted entity.

**Handling**: Allow correction, mark entity as retroactively deleted:

```typescript
// Entity was deleted
// But we need to correct a field retroactively

await retroactiveCorrector.correct({
  entity_id: "deleted_entity_001",
  field_name: "amount",
  old_value: 100.00,
  new_value: 105.00,
  effective_date: "2025-01-15T10:00:00Z",
  reason: "Correcting amount before deletion",
  user_id: "user_jane"
});

// Correction recorded in provenance ledger
// Entity remains deleted but historical value corrected
```

---

## Performance Characteristics

### Latency Targets

| Operation | Complexity | Target Latency (p95) | Notes |
|-----------|-----------|---------------------|-------|
| `correct()` | Simple | < 200ms | Single entity, no approval |
| `correct()` | Complex | < 500ms | With impact analysis |
| `validateRetroactiveChange()` | Any | < 100ms | Validation only |
| `getImpactAnalysis()` | < 10 entities | < 200ms | Simple dependency tree |
| `getImpactAnalysis()` | 10-100 entities | < 1s | Complex dependencies |
| `requestApproval()` | Any | < 100ms | Create approval record |
| `approve()` | Any | < 300ms | Apply correction + update |

### Throughput

- **Simple Corrections**: 100-500 corrections/sec
- **Complex Corrections**: 10-50 corrections/sec (with impact analysis)

---

## Implementation Notes

### TypeScript Implementation

```typescript
export class RetroactiveCorrector {
  constructor(
    private provenanceLedger: ProvenanceLedger,
    private bitemporalQuery: BitemporalQuery,
    private authorizationService: AuthorizationService
  ) {}

  async correct(
    correction: RetroactiveCorrection
  ): Promise<CorrectionResult> {
    // Implementation shown in Core Functionality
    // ...
  }

  async validateRetroactiveChange(
    correction: RetroactiveCorrection
  ): Promise<ValidationResult> {
    // Implementation shown in Core Functionality
    // ...
  }

  async getImpactAnalysis(
    correction: RetroactiveCorrection
  ): Promise<ImpactAnalysis> {
    const affected: AffectedEntity[] = [];

    // Find entities that reference this entity
    const references = await this.findReferences(correction.entity_id);

    for (const ref of references) {
      affected.push({
        entity_id: ref.entity_id,
        entity_type: ref.entity_type,
        relationship: ref.relationship_type,
        impact_type: this.determineImpactType(correction.field_name, ref)
      });
    }

    // Find calculations that use this field
    const recalc = await this.findCalculations(
      correction.entity_id,
      correction.field_name
    );

    // Assess severity
    const severity = this.assessSeverity(affected.length, correction.field_name);

    return {
      entity_id: correction.entity_id,
      field_name: correction.field_name,
      value_change: {
        from: correction.old_value,
        to: correction.new_value
      },
      affected_entities: affected,
      recalculation_needed: recalc,
      workflows_to_rerun: [],
      impact_severity: severity,
      estimated_affected_count: affected.length
    };
  }

  private assessSeverity(
    affectedCount: number,
    fieldName: string
  ): 'low' | 'medium' | 'high' | 'critical' {
    const criticalFields = ['amount', 'revenue', 'cost', 'premium'];

    if (affectedCount > 100 || criticalFields.includes(fieldName)) {
      return 'critical';
    } else if (affectedCount > 10) {
      return 'high';
    } else if (affectedCount > 1) {
      return 'medium';
    } else {
      return 'low';
    }
  }
}
```

---

## Audit & Compliance

### Audit Trail Components

Every correction creates:

1. **Provenance Event**: In provenance ledger
2. **Correction Record**: In corrections table
3. **Approval Record**: If approval required
4. **Impact Report**: Stored with correction

### Compliance Reporting

```typescript
async function generateComplianceReport(
  startDate: string,
  endDate: string
): Promise<ComplianceReport> {
  const corrections = await retroactiveCorrector.getCorrectionHistory();

  const filtered = corrections.filter(
    c => c.applied_at >= startDate && c.applied_at <= endDate
  );

  return {
    period: { start: startDate, end: endDate },
    total_corrections: filtered.length,
    by_field: groupBy(filtered, c => c.field_name),
    by_user: groupBy(filtered, c => c.user_id),
    by_severity: groupBy(filtered, c => c.impact_severity),
    approvals_required: filtered.filter(c => c.requires_approval).length,
    approvals_granted: filtered.filter(c => c.status === 'applied').length
  };
}
```

---

## Integration Patterns

### Pattern 1: UI Correction Workflow

```typescript
class CorrectionUI {
  async initiateCorrection(
    entityId: string,
    fieldName: string,
    newValue: any
  ) {
    // 1. Get current value
    const current = await getCurrentFieldValue(entityId, fieldName);

    // 2. Validate
    const validation = await retroactiveCorrector.validateRetroactiveChange({
      entity_id: entityId,
      field_name: fieldName,
      old_value: current,
      new_value: newValue,
      effective_date: getCurrentEffectiveDate(entityId),
      reason: await promptUserForReason(),
      user_id: getCurrentUser()
    });

    if (!validation.is_valid) {
      showErrors(validation.errors);
      return;
    }

    // 3. Show impact analysis
    const impact = await retroactiveCorrector.getImpactAnalysis({...});
    const proceed = await confirmWithUser(impact);

    if (!proceed) return;

    // 4. Apply correction
    const result = await retroactiveCorrector.correct({...});

    showSuccess(result);
  }
}
```

---

## Multi-Domain Examples

(Already covered extensively in Multi-Domain Applicability section with 7 domains)

---

## Testing Strategy

### Unit Tests

```typescript
describe('RetroactiveCorrector', () => {
  let corrector: RetroactiveCorrector;

  beforeEach(() => {
    corrector = new RetroactiveCorrector(
      provenanceLedger,
      bitemporalQuery,
      authService
    );
  });

  describe('correct()', () => {
    it('should apply valid correction', async () => {
      const result = await corrector.correct({
        entity_id: 'txn_001',
        field_name: 'amount',
        old_value: 100.00,
        new_value: 105.00,
        effective_date: '2025-01-15T10:00:00Z',
        reason: 'Test correction',
        user_id: 'user_test'
      });

      expect(result.status).toBe('applied');
      expect(result.provenance_record_id).toBeDefined();
    });

    it('should reject invalid correction', async () => {
      await expect(
        corrector.correct({
          entity_id: 'txn_001',
          field_name: 'amount',
          old_value: 100.00,
          new_value: 100.00, // Same value
          effective_date: '2025-01-15T10:00:00Z',
          reason: 'Test',
          user_id: 'user_test'
        })
      ).rejects.toThrow('Validation failed');
    });
  });

  describe('getImpactAnalysis()', () => {
    it('should identify affected entities', async () => {
      const impact = await corrector.getImpactAnalysis({
        entity_id: 'txn_001',
        field_name: 'amount',
        old_value: 100.00,
        new_value: 105.00,
        effective_date: '2025-01-15T10:00:00Z',
        reason: 'Test',
        user_id: 'user_test'
      });

      expect(impact.affected_entities.length).toBeGreaterThan(0);
      expect(impact.impact_severity).toBeDefined();
    });
  });
});
```

---

## Migration Guide

### From Manual Corrections

```typescript
// Before: Manual update without audit trail
await db.query(`
  UPDATE transactions
  SET amount = 105.00
  WHERE id = 'txn_001'
`);

// After: Retroactive correction with full audit trail
await retroactiveCorrector.correct({
  entity_id: "txn_001",
  field_name: "amount",
  old_value: 100.00,
  new_value: 105.00,
  effective_date: "2025-01-15T10:00:00Z",
  reason: "Corrected from receipt",
  user_id: currentUser.id
});
```

---

## Related Primitives

- **ProvenanceLedger**: Stores correction events
- **BitemporalQuery**: Queries historical state
- **TimelineReconstructor**: Visualizes correction timeline
- **ValidationEngine**: Additional validation rules

---

## References

- [ProvenanceLedger Primitive](./ProvenanceLedger.md)
- [BitemporalQuery Primitive](./BitemporalQuery.md)
- [TimelineReconstructor Primitive](./TimelineReconstructor.md)
- [Bitemporal Data Management](https://en.wikipedia.org/wiki/Temporal_database)

---

**End of RetroactiveCorrector OL Primitive Specification**
