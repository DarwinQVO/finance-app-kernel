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

## Related Primitives

- **Normalizer**: Execution is logged by NormalizationLog
- **UploadRecord**: References NormalizationLog via `normalization_log_ref`
- **ProvenanceLedger**: Records normalize events using data from NormalizationLog

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
