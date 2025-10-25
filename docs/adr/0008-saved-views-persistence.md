# ADR-0008: Saved Views Persistence Strategy

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Dashboard configuration persistence

---

## Context

Users need to save custom dashboard configurations for reuse across sessions and devices:

```
User creates view: "Q4 2024 Expenses" (filters + layout + metrics)
  → Switches device/browser
  → Expects same view available
```

**Key requirements:**

1. **Cross-device sync**: View available on desktop, mobile, tablet
2. **Shareable**: Users can share view configs via URL
3. **Versioned**: Schema evolution without breaking old configs
4. **Audit trail**: Track who created/modified views

**Configuration includes:**
- **Filters**: Date range, categories, counterparties, amount range
- **Layout**: Chart types, column visibility, sort order
- **Metrics**: KPIs, aggregations, comparison periods

---

## Decision

**We persist saved views as JSON config in database with versioned schema.**

View types:
1. **System views** (read-only, predefined): "All Transactions", "This Month", "Top Expenses"
2. **User views** (user-created, editable): "Q4 2024 Expenses", "Travel Budget"
3. **Shared views** (shareable via link): `GET /views/shared/{share_token}`

---

## Rationale

### 1. Cross-Device Synchronization
- Database persistence syncs across all user devices
- No dependency on browser localStorage (device-specific)
- Works with mobile apps, web, desktop

### 2. Shareable Configurations
- Generate share token: `/views/shared/VW_abc123`
- Recipient gets read-only copy of view
- Use cases: Team dashboards, client reports

### 3. Schema Versioning
- JSON config includes `schema_version: "1.0"`
- Migrations handle old configs gracefully
- Example: v1.0 → v2.0 adds new filter types

### 4. Audit Trail
- Track `created_by`, `created_at`, `modified_at`
- Compliance: Who configured financial reports?
- Debugging: When did view break?

---

## Consequences

### ✅ Positive

- **Cross-device sync**: View available everywhere user logs in
- **Shareable**: Team collaboration via shared links
- **Version control**: Schema evolution support
- **Audit trail**: Compliance and debugging

### ⚠️ Trade-offs

- **More storage**: ~1KB per view (acceptable for 1000s of views)
- **Schema versioning complexity**: Need migration logic for config changes
- **Query overhead**: Must join `saved_views` table for user dashboard
- **Sync latency**: Changes take ~1s to sync across devices (acceptable)

### 🔴 Risks (Mitigated)

- **Risk**: Schema migration breaks old views
  - **Mitigation**: Backwards-compatible migrations with fallbacks
  - Example: Unknown filter type → ignored, not error

- **Risk**: Shared views expose sensitive data
  - **Mitigation**: Share tokens expire after 30 days
  - User can revoke share token anytime

---

## Alternatives Considered

### Alternative A: URL Parameters Only (Rejected)

**Pros:**
- Simple: No database storage
- Fast: No server round-trip
- Shareable: URL contains full config

**Cons:**
- **Not persistent**: Lose view when URL lost
- **URL length limits**: Max 2048 chars (complex views exceed this)
- **No sync**: Different URL on each device
- **No versioning**: Cannot migrate old URLs

### Alternative B: Browser localStorage Only (Rejected)

**Pros:**
- Fast: Instant access, no server latency
- Zero server load: All client-side

**Cons:**
- **Device-specific**: Not synced across devices
- **Not shareable**: Cannot send view to others
- **No backup**: Lost if user clears browser data
- **No audit trail**: Cannot track who created view

### Alternative C: Hardcoded Views Only (Rejected)

**Pros:**
- Zero storage overhead
- Zero schema versioning complexity
- Predictable: All users see same views

**Cons:**
- **Inflexible**: Cannot customize for user needs
- **No personalization**: Every user sees same dashboards
- **No collaboration**: Cannot share custom analyses

---

## Implementation Notes

### Schema Structure

```json
{
  "view_id": "VW_abc123",
  "schema_version": "1.0",
  "name": "Q4 2024 Expenses",
  "type": "user",
  "created_by": "user_123",
  "created_at": "2025-10-24T12:00:00Z",
  "modified_at": "2025-10-24T14:30:00Z",
  "config": {
    "filters": {
      "date_range": { "start": "2024-10-01", "end": "2024-12-31" },
      "categories": ["groceries", "dining", "transport"],
      "amount_min": 10.00
    },
    "layout": {
      "chart_type": "bar",
      "columns": ["date", "merchant", "amount", "category"],
      "sort_by": "amount",
      "sort_order": "desc"
    },
    "metrics": {
      "kpis": ["total_spent", "avg_transaction", "transaction_count"],
      "compare_to": "previous_quarter"
    }
  },
  "share_token": null
}
```

### Database Schema

```sql
CREATE TABLE saved_views (
  view_id VARCHAR(50) PRIMARY KEY,
  user_id VARCHAR(50) NOT NULL,
  name VARCHAR(255) NOT NULL,
  type VARCHAR(20) NOT NULL, -- 'system' | 'user' | 'shared'
  schema_version VARCHAR(10) NOT NULL,
  config JSONB NOT NULL,
  share_token VARCHAR(50) UNIQUE,
  share_expires_at TIMESTAMP,
  created_at TIMESTAMP NOT NULL,
  modified_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_saved_views_user ON saved_views (user_id);
CREATE INDEX idx_saved_views_share ON saved_views (share_token)
WHERE share_token IS NOT NULL;
```

### Migration Example (v1.0 → v2.0)

```javascript
function migrateViewConfig(config) {
  if (config.schema_version === "1.0") {
    // v2.0 adds new filter: "exclude_internal_transfers"
    config.filters.exclude_internal_transfers = true; // default
    config.schema_version = "2.0";
  }
  return config;
}
```

---

## Related Decisions

- **ADR-0006**: Pagination applies to filtered transactions in saved views
- **Future**: May add view templates (e.g., "Budget Tracker Template")
- **Future**: May add collaborative editing for shared views

---

**References:**
- Vertical 2.3: [docs/verticals/2.3-finance-dashboard.md](../verticals/2.3-finance-dashboard.md)
- SavedViewStore: [docs/primitives/ol/SavedViewStore.md](../primitives/ol/SavedViewStore.md)
- Schema: [docs/schemas/saved-view.schema.json](../schemas/saved-view.schema.json)
