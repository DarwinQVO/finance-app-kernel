# OL Primitive: NormalizationLog

**Type**: Observability / Structured Log
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.3 (Normalization)

---

## Purpose

Structured execution log for normalization operations. Records validation results (successes, failures, warnings), performance metrics, and applied rules. Enables debugging, quality monitoring, and audit trails for normalization pipelines.

---

## Schema

See [`normalization-log.schema.json`](../../schemas/normalization-log.schema.json) for complete schema.

**Key fields:**
```typescript
interface NormalizationLog {
  upload_id: string
  normalizer_version: string
  rule_set_version: string
  observations_processed: number
  canonicals_created: number
  failures: number
  warnings: number
  failure_details: Array<{observation_id, field, error}>
  warning_details: Array<{canonical_id, flag, message}>
  duration_ms: number
  started_at: ISO8601DateTime
  completed_at: ISO8601DateTime
}
```

---

## Behavior

**Success case (all observations normalized):**
```json
{
  "upload_id": "UL_abc123",
  "normalizer_version": "1.0.0",
  "rule_set_version": "2024.10",
  "observations_processed": 100,
  "canonicals_created": 100,
  "failures": 0,
  "warnings": 8,
  "duration_ms": 5120,
  "warning_details": [
    {"canonical_id": "CT_xyz", "flag": "possible_duplicate", "message": "Similar transaction found"}
  ]
}
```

**Partial success (some observations failed):**
```json
{
  "upload_id": "UL_def456",
  "normalizer_version": "1.0.0",
  "rule_set_version": "2024.10",
  "observations_processed": 100,
  "canonicals_created": 95,
  "failures": 5,
  "warnings": 12,
  "failure_details": [
    {"observation_id": "UL_def456:42", "field": "date", "error": "Unparseable date format"}
  ],
  "duration_ms": 6200
}
```

---

## Storage

**Path:** `/logs/normalize/{upload_id}.log.json`
**Persistence:** Write-once, immutable

---

## Multi-Domain Applicability

**Finance:** Log transaction normalization (validation failures, duplicate rate)
**Healthcare:** Log lab result normalization (unit conversion errors, range violations)
**Legal:** Log clause normalization (categorization accuracy, parsing issues)
**Research (RSRCH - Utilitario):** Log fact normalization (entity name resolution errors, missing source credibility, '@sama' → 'Sam Altman' transformations)
**Manufacturing:** Log measurement normalization (calibration errors, out-of-range values)

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Log transaction normalization execution for quality monitoring
**Example:** Normalizer processes 100 raw transactions → NormalizationLog records {upload_id: "UL_abc123", observations_processed: 100, canonicals_created: 100, failures: 0, warnings: 8, duration_ms: 5120, warning_details: [{canonical_id: "CT_xyz", flag: "possible_duplicate"}]} → Quality dashboard queries logs → Calculates normalization success rate (100%), warning rate (8%), p95 latency (6.1s) → Alert if failures > 5%
**Operations:** Write immutable log, query for quality metrics (success rate, latency, failure types), track warning trends (duplicate rate increasing = potential data quality issue)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Log lab result normalization for error tracking and compliance
**Example:** Normalizer processes 50 lab observations → 45 succeed, 5 fail → NormalizationLog records {observations_processed: 50, canonicals_created: 45, failures: 5, failure_details: [{observation_id: "obs_42", field: "unit", error: "Unknown unit 'mmol' for test 'Glucose'"}]} → Admin checks log → Sees unit conversion errors → Adds mmol→mg/dL conversion rule
**Status:** ✅ Conceptually validated

### ✅ Legal
**Use case:** Log clause normalization for audit trail (compliance requirement)
**Example:** Normalizer categorizes 25 contract clauses → NormalizationLog records {observations_processed: 25, canonicals_created: 25, rule_set_version: "2024.10", warnings: 3, warning_details: [{canonical_id: "clause_12", flag: "low_confidence", message: "Clause category unclear (0.65 confidence)"}]} → Audit requires proof of categorization → Log provides immutable record with rule version, timestamps
**Status:** ✅ Conceptually validated

### ✅ RSRCH (Utilitario Research)
**Use case:** Log fact normalization for entity resolution tracking and data quality
**Example:** Normalizer processes 50 raw founder facts from TechCrunch → Resolves entity names → NormalizationLog records {upload_id: "UL_tc_123", observations_processed: 50, canonicals_created: 48, failures: 2, warnings: 12, failure_details: [{observation_id: "obs_23", field: "subject_entity", error: "Cannot resolve 'Sammy A.' to canonical founder name"}], warning_details: [{canonical_id: "fact_5", flag: "low_confidence_entity_match", message: "'Sam Altman' → '@sama' confidence 0.72"}], duration_ms: 3200} → Dashboard tracks entity resolution accuracy (96% success rate), low confidence matches (24% of facts), normalization latency (p95: 4.5s)
**Operations:** Logs track entity resolution quality (success rate, confidence scores), failure details guide manual review (unresolved entities queued for human labeling), warning trends signal data quality issues (low confidence matches increasing = need better entity rules)
**Status:** ✅ Conceptually validated

### ✅ E-commerce
**Use case:** Log product normalization for catalog quality monitoring
**Example:** Normalizer processes 1,500 product observations → 1,480 succeed, 20 fail → NormalizationLog records {observations_processed: 1500, canonicals_created: 1480, failures: 20, failure_details: [{observation_id: "prod_342", field: "category", error: "Invalid category 'Electronics > Phones > iPhone Pro Max' (too many levels)"}]} → Alert fires if failure rate > 2% → Admin reviews failed products, fixes category taxonomy
**Status:** ✅ Conceptually validated

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (structured execution log for validation pipeline, no domain-specific code)
**Reusability:** High (same log schema works for transaction normalization, lab test normalization, clause categorization, fact entity resolution, product categorization; only failure/warning types differ)

---

## Related Primitives

- **Normalizer**: Execution is logged by NormalizationLog
- **UploadRecord**: References NormalizationLog via `normalization_log_ref`
- **ProvenanceLedger**: Records normalize events using data from NormalizationLog

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
