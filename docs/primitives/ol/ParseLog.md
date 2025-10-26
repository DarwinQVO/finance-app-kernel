# OL Primitive: ParseLog

**Type**: Observability / Structured Log
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.2 (Extraction)

---

## Purpose

Structured execution log for parser operations. Records what happened during parsing: success/failure, rows extracted, execution time, warnings, errors. Enables debugging, monitoring, and audit trails.

---

## Schema

See [`parse-log.schema.json`](../../schemas/parse-log.schema.json) for complete schema.

**Key fields:**
```typescript
interface ParseLog {
  upload_id: string
  parser_id: string
  parser_version: string
  success: boolean
  rows_extracted: number
  duration_ms: number
  warnings: string[]
  errors: string[]
  started_at: ISO8601DateTime
  completed_at: ISO8601DateTime
}
```

---

## Behavior

**Success case:**
```json
{
  "upload_id": "UL_abc123",
  "parser_id": "bofa_pdf_parser",
  "parser_version": "1.2.0",
  "success": true,
  "rows_extracted": 42,
  "duration_ms": 3421,
  "warnings": ["Row 15: Ambiguous date format"],
  "errors": [],
  "started_at": "2025-10-23T14:35:00Z",
  "completed_at": "2025-10-23T14:35:03Z"
}
```

**Failure case:**
```json
{
  "upload_id": "UL_def456",
  "parser_id": "bofa_pdf_parser",
  "parser_version": "1.2.0",
  "success": false,
  "rows_extracted": 0,
  "duration_ms": 150,
  "warnings": [],
  "errors": ["PDF structure invalid", "Cannot extract table"],
  "started_at": "2025-10-23T14:40:00Z",
  "completed_at": "2025-10-23T14:40:00Z"
}
```

---

## Storage

**Path:** `/logs/parse/{upload_id}.log.json`

**Persistence:** Write-once, immutable. Never update existing log.

---

## Multi-Domain Applicability

**Finance:** Log bank statement parsing (success rate, extraction latency)
**Healthcare:** Log lab result extraction (which tests extracted, errors)
**Legal:** Log contract parsing (clauses found, parsing time)
**Research (RSRCH - Utilitario):** Log fact extraction from web pages (facts found, parsing errors, source credibility score)
**Manufacturing:** Log sensor data parsing (data points extracted, corruption detected)

---

## Related Primitives

- **Parser**: Execution is logged by ParseLog
- **UploadRecord**: References ParseLog via `parse_log_ref`
- **ProvenanceLedger**: Records parse events using data from ParseLog

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
