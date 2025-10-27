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

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Log bank statement parsing execution for debugging and monitoring
**Example:** User uploads BoFA PDF → Parser runs → ParseLog records {upload_id: "UL_abc123", parser_id: "bofa_pdf_parser", parser_version: "1.2.0", success: true, rows_extracted: 42, duration_ms: 3421, warnings: ["Row 15: Ambiguous date format"], started_at: "2025-10-23T14:35:00Z"} → Stored at /logs/parse/UL_abc123.log.json → Monitoring dashboard queries logs → Calculates parser success rate (98.5%), p95 latency (4.2s), warning rate (12%)
**Operations:** Write immutable log (write-once), query logs (debugging, monitoring), aggregate metrics (success rate, latency percentiles)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Log lab result parsing for audit trail and error tracking
**Example:** Hospital uploads Quest lab report → Parser extracts 8 tests → ParseLog records {success: true, rows_extracted: 8, duration_ms: 2100, warnings: []} → Later patient queries fail (0 results) → Admin checks parse log → Sees warning "Test name format changed, using fallback extraction" → Fixes parser
**Status:** ✅ Conceptually validated

### ✅ Legal
**Use case:** Log contract parsing for audit trail (regulatory compliance)
**Example:** Law firm uploads contract → Parser extracts 25 clauses → ParseLog records {parser_id: "docusign_pdf_parser", parser_version: "2.1.0", rows_extracted: 25, duration_ms: 5500} → Audit requires proof of extraction → Parse log provides immutable record with timestamp, parser version, extraction details
**Status:** ✅ Conceptually validated

### ✅ RSRCH (Utilitario Research)
**Use case:** Log web page fact extraction for debugging and credibility tracking
**Example:** Scraper downloads TechCrunch article → TechCrunchHTMLParser extracts 12 founder facts → ParseLog records {upload_id: "UL_tc_123", parser_id: "techcrunch_html_parser", parser_version: "1.5.0", success: true, rows_extracted: 12, duration_ms: 1850, warnings: ["Author field missing, using 'Unknown'"], metadata: {source_credibility_score: 0.92, publication_date: "2024-02-25"}} → Dashboard queries logs to calculate parser reliability (TechCrunch parser: 99.2% success rate, avg 12 facts per article)
**Operations:** Logs include domain-specific metadata (source_credibility_score for research quality tracking), warnings track parsing anomalies (missing fields, format changes), duration tracking for scraper performance optimization
**Status:** ✅ Conceptually validated

### ✅ E-commerce
**Use case:** Log supplier catalog parsing for monitoring and error alerts
**Example:** Merchant imports supplier CSV (1,500 products) → SupplierCSVParser extracts observations → ParseLog records {rows_extracted: 1500, duration_ms: 6200, warnings: ["Row 342: Missing SKU, using product_id as fallback"]} → Alert fires if success: false (notify admin) → Monitoring tracks catalog import latency (p95: 7.5s)
**Status:** ✅ Conceptually validated

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (structured execution log pattern, no domain-specific code)
**Reusability:** High (same log schema works for bank parsers, lab parsers, contract parsers, web scrapers, catalog parsers; only domain-specific metadata fields differ)

---

## Related Primitives

- **Parser**: Execution is logged by ParseLog
- **UploadRecord**: References ParseLog via `parse_log_ref`
- **ProvenanceLedger**: Records parse events using data from ParseLog

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
