# ADR-0039: API Versioning Strategy

**Status:** ✅ Accepted
**Date:** 2025-10-25
**Vertical:** 5.5 Public API Contracts
**Deciders:** Architecture Team, Product Team, Developer Experience Team

---

## Context

Our public REST API will evolve over time as we add features, fix bugs, and improve performance. We need a versioning strategy that:

1. **Maintains backward compatibility** for existing integrations (breaking changes are costly for customers)
2. **Allows controlled evolution** without blocking new features
3. **Provides clear migration paths** when breaking changes are necessary
4. **Minimizes operational complexity** (don't maintain too many versions simultaneously)
5. **Follows industry best practices** for API versioning

**Key Questions:**
- How do we communicate versions? (URL path, header, query param?)
- When do we increment versions? (major vs minor vs patch?)
- How long do we support old versions? (deprecation policy?)
- Can we have multiple versions active simultaneously?

---

## Decision

We will use **URL path versioning** with **semantic versioning** semantics and a **12-month sunset policy**.

### 1. Version Format

**URL Path Versioning:**
```
https://api.truthconstruction.com/v1/observations
https://api.truthconstruction.com/v2/observations (future)
```

**Why URL path (not header or query param)?**
- ✅ **Most discoverable:** Version visible in browser address bar, curl commands, API logs
- ✅ **Cache-friendly:** CDN/proxy can cache different versions separately
- ✅ **Industry standard:** Used by Stripe, GitHub, Twilio, most major APIs
- ✅ **Simple routing:** Easy to route v1 → service_v1, v2 → service_v2 in API gateway
- ❌ Header versioning (Accept: application/vnd.api+json; version=1) is less discoverable
- ❌ Query param (?version=1) can conflict with filtering params, less idiomatic

### 2. Version Increment Rules

**Major version (v1 → v2):** Breaking changes only
- Changed response structure (removed/renamed fields)
- Changed request validation (required field added, enum values removed)
- Changed authentication (API key format changed)
- Changed semantics (same endpoint, different behavior)

**Minor version (within v1):** Additive changes, no version bump
- Added optional request fields
- Added response fields (existing fields unchanged)
- New endpoints (e.g., POST /v1/reconciliations/bulk)
- Performance improvements, bug fixes

**Example:**
```
# v1 (original)
GET /v1/observations
Response: {"observations": [...], "total": 42}

# v1 (additive, no version bump)
GET /v1/observations
Response: {"observations": [...], "total": 42, "has_more": true}  # Added has_more field

# v2 (breaking change, new version)
GET /v2/observations
Response: {"data": [...], "metadata": {"total": 42, "has_more": true}}  # Renamed observations → data
```

### 3. Deprecation Policy

**Timeline:**
1. **Deprecation Notice (T+0):** 6 months advance warning
   - Email all affected tenants (using deprecated version)
   - Changelog entry with migration guide
   - API response header: `Deprecation: true`, `Sunset: 2026-04-25`
   - Dashboard banner: "API v1 will be sunset on 2026-04-25. Migrate to v2."

2. **Deprecation Period (T+0 to T+6 months):** Both versions active
   - v1 still works (no breaking changes)
   - v2 available for migration
   - Documentation shows v2 as "current", v1 as "deprecated"
   - Support prioritizes v2 issues

3. **Sunset Notice (T+6 months):** Final warnings
   - Email all remaining v1 users (monthly reminders)
   - API response header: `Warning: "API v1 will be sunset in 30 days"`
   - Dashboard banner: "URGENT: Migrate to v2 by 2026-04-25"

4. **Sunset (T+12 months):** v1 returns HTTP 410 Gone
   - v1 endpoints return: `HTTP 410 Gone`
   - Response body: `{"error": "API v1 has been sunset. Please migrate to v2: https://docs.example.com/v2-migration"}`
   - v2 continues to work

**Why 12 months?**
- Enterprise customers need time to plan migrations (quarterly release cycles, compliance reviews)
- 12 months = 4 quarterly releases (adequate for most organizations)
- Shorter (6 months) is too aggressive for large customers
- Longer (18+ months) increases maintenance burden

### 4. Parallel Version Support

**Active Versions:**
- **v1 (current stable):** Fully supported, receives bug fixes + security patches
- **v2 (future, during deprecation period):** Fully supported, new features added here
- **v0 (legacy, if exists):** Sunset (returns HTTP 410 Gone)

**Max Parallel Versions:** 2 (current + deprecated)
- Never support v1, v2, v3 simultaneously (maintenance nightmare)
- Customers must migrate v1 → v2 before v3 is released

---

## Consequences

### Positive

✅ **Clear version visibility:** URL path makes version obvious in logs, docs, curl commands

✅ **Backward compatibility:** Existing v1 integrations continue working during 12-month deprecation

✅ **Controlled evolution:** Breaking changes gated behind major version bump

✅ **Migration time:** 12-month window gives customers adequate time to migrate

✅ **Reduced maintenance:** Max 2 versions simultaneously (not 3, 4, 5...)

✅ **Industry alignment:** Follows best practices from Stripe, GitHub, Twilio

### Negative

❌ **URL duplication:** Same endpoint at /v1/observations and /v2/observations (code duplication risk)

❌ **Migration effort:** Customers must update all API calls (change URLs from v1 → v2)

❌ **Deprecation communication:** Requires proactive customer communication (emails, banners, docs)

❌ **Sunset enforcement:** Must build HTTP 410 handler, monitor v1 usage, enforce sunset date

### Mitigations

**Code duplication:**
- Share business logic between v1 and v2 handlers (only serialization differs)
- Use adapter pattern: `v1_handler = adapt_v2_to_v1(v2_handler)`

**Migration friction:**
- Provide migration scripts/tools (e.g., regex search-replace for URL patterns)
- Offer migration consulting for Enterprise customers (dedicated support)
- Auto-generate v2 client SDKs from OpenAPI spec

**Deprecation enforcement:**
- Automated emails (30 days before sunset, weekly reminders)
- Dashboard metrics: "% of requests using deprecated versions"
- Support team trained to guide customers through migration

---

## Alternatives Considered

### Alternative 1: Header-Based Versioning

**Example:**
```
GET /observations
Accept: application/vnd.truthconstruction.v1+json
```

**Pros:**
- Cleaner URLs (no /v1/ prefix)
- Supports content negotiation (JSON vs XML)

**Cons:**
- ❌ Less discoverable (version hidden in headers)
- ❌ Not cache-friendly (CDN must parse headers)
- ❌ More complex routing (API gateway must inspect headers)
- ❌ Uncommon in modern APIs (Stripe, GitHub, Twilio use URL path)

**Decision:** ❌ Rejected - URL path is more discoverable and follows industry standards

### Alternative 2: Query Parameter Versioning

**Example:**
```
GET /observations?version=1
```

**Pros:**
- No URL structure changes (same path for all versions)

**Cons:**
- ❌ Can conflict with filtering params (e.g., /observations?version=1&type=transaction)
- ❌ Less idiomatic (uncommon in REST APIs)
- ❌ Version not visible in base URL (curl/logs show /observations, not /observations?version=1)

**Decision:** ❌ Rejected - URL path is more idiomatic and avoids query param conflicts

### Alternative 3: No Versioning (Rolling Updates)

**Example:**
- Always deploy changes to same endpoints
- Break compatibility when necessary

**Pros:**
- No version management complexity
- Simpler deployment (single version)

**Cons:**
- ❌ **Breaking changes break all customers simultaneously** (catastrophic)
- ❌ No migration path (customers forced to upgrade immediately)
- ❌ Difficult rollback (can't roll back if customers depend on new behavior)

**Decision:** ❌ Rejected - Unacceptable for public API (breaks customer trust)

### Alternative 4: Infinite Version Support

**Example:**
- Support v1, v2, v3, v4, ... indefinitely

**Pros:**
- No forced migrations (customers choose when to upgrade)

**Cons:**
- ❌ **Unsustainable maintenance burden** (security patches for v1, v2, v3, v4...)
- ❌ Slows feature development (must test all versions)
- ❌ Increases operational complexity (more deployments, more monitoring)

**Decision:** ❌ Rejected - Maintenance burden outweighs benefits

---

## Related Decisions

- **ADR-0040 (API Stability Tiers):** Defines stability guarantees (stable, beta, alpha)
- **ADR-0041 (Webhook Versioning):** Webhooks follow same versioning as REST API

---

## References

- [Stripe API Versioning](https://stripe.com/docs/api/versioning)
- [GitHub API Versioning](https://docs.github.com/en/rest/overview/api-versioning)
- [RFC 7231 (HTTP Semantics)](https://tools.ietf.org/html/rfc7231)
- [API Evolution Patterns](https://martinfowler.com/articles/richardsonMaturityModel.html)
