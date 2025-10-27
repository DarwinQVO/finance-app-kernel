# Audit System (Provenance)

**Status**: Universal / Domain-Agnostic
**Purpose**: Provide bitemporal audit trail and temporal query capabilities for all systems

---

## Overview

The Audit System is an independent, append-only event store that tracks the complete history of all operations across all systems. It enables answering questions like:
- "What did we know on January 15th?" (transaction_time query)
- "What was true on January 15th?" (valid_time query)
- "Show me the complete timeline of this entity"
- "Perform a retroactive correction and recalculate downstream effects"

## Architecture

```
Events from Any System → ProvenanceLedger → Temporal Queries + Visualizations
```

### Core Concept: Bitemporal Tracking

Every event tracked in two dimensions:
- **transaction_time**: When we recorded the event (immutable)
- **valid_time**: When the event was actually true in the real world (can be backdated)

---

## Primitives

### Event Storage
- [ProvenanceLedger](primitives/ProvenanceLedger.md) - Append-only bitemporal event store
- [AuditLog](primitives/AuditLog.md) - Structured logging for compliance

### Temporal Queries
- [BitemporalQuery](primitives/BitemporalQuery.md) - Query events by transaction_time or valid_time

### Visualizations
- [TimelineReconstructor](primitives/TimelineReconstructor.md) - Generate timelines showing "what we knew when"

### Corrections
- [RetroactiveCorrector](primitives/RetroactiveCorrector.md) - Handle retroactive corrections with full audit trail

---

## Multi-Domain Applicability

The Audit System works identically across domains:

| Domain | Events Tracked | Use Case |
|--------|----------------|----------|
| **Finance** | Transaction created, updated, deleted | "Show all changes to transaction #42" |
| **Healthcare** | Lab result recorded, amended | "What was the diagnosis on March 1st?" |
| **Legal** | Contract signed, amended, terminated | "Show contract lifecycle timeline" |
| **Research (RSRCH)** | Fact extracted, validated, corrected | "Track fact provenance from source to truth" |
| **E-commerce** | Product created, price changed | "Audit price changes for product #123" |
| **Manufacturing** | Sensor reading recorded, calibration applied | "Show sensor history with corrections" |

---

## Key Features

### 1. Append-Only
- Events are NEVER deleted or modified
- Corrections add new events, preserving original records

### 2. Bitemporal Tracking
- transaction_time: When we recorded the event (system time)
- valid_time: When the event was true (business time)
- Enables "time travel" queries: "What did we know on date X?"

### 3. Event Sourcing
- All systems log operations → ProvenanceLedger
- Complete audit trail for compliance and debugging

### 4. Retroactive Corrections
- Correct past mistakes without losing history
- Recalculate downstream effects automatically

---

## Integration with Other Systems

### Truth Construction System
- **Upload** logs: file_uploaded, parse_started, parse_completed
- **Extract** logs: observations_extracted, parser_version
- **Normalize** logs: observations_validated, canonical_records_created
- **Query** logs: query_executed, export_generated

### API/Auth System
- **Authentication** logs: user_logged_in, token_issued
- **Authorization** logs: access_granted, access_denied
- **API** logs: request_received, response_sent

### Finance App
- **Domain operations** logs: account_created, counterparty_merged, relationship_established

---

## Performance Optimizations (P1 Fixes Complete)

✅ **Streaming API**: Process millions of events without OOM (memory O(batchSize) instead of O(total_results))
✅ **Batch Queries**: 31x faster for multi-entity queries
✅ **explainPlan()**: Debug slow queries with index usage analysis

---

## Key Principles

1. **Independent**: No dependencies on other systems (but used BY all systems)
2. **Append-Only**: Immutable event history
3. **Bitemporal**: Track when we knew it AND when it was true
4. **Universal Pattern**: Works for any domain, any event type

---

## Related Documentation

- [Truth Construction System](../truth-construction/README.md) - Primary user of Audit System
- [API/Auth System](../api-auth/README.md) - Logs authentication/authorization events
- [Finance App Example](../../verticals/README.md) - Domain-specific event tracking

---

**Primitives Count**: 5
**Status**: Production-ready (all primitives include P1 optimizations)
**Last Updated**: 2025-10-27
