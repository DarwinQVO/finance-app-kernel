# Phase 5: Refactor Primitives to Literate Style

**Status:** In Progress
**Date Started:** 2025-10-27
**Context:** Phases 1-4 complete, need to apply narrative style to primitive docs
**Goal:** Expand Simplicity Profiles from 1-line to full literate programming style

---

## What is "Literate Style"?

**Current state (Phase 2):**
```markdown
## Simplicity Profiles

**Personal - ~15 LOC:** Single hardcoded API key check
**Small Business - ~60 LOC:** SQLite key store, rate limiting
**Enterprise - ~300 LOC:** PostgreSQL, key rotation, scopes, audit
```

**Target state (Phase 5 - Literate):**
```markdown
## Simplicity Profiles

### Personal Profile (15 LOC)

**Contexto del Usuario:**
Darwin tiene una app personal de finanzas. Genera 1 API key para su script
de subida automática de estados de cuenta. El key nunca expira, no hay
multi-tenancy, no hay scopes (full access).

**Implementation:**
```python
# api_auth.py (Personal - 15 LOC)
API_KEY = "my_secret_key_abc123"  # Hardcoded (YAGNI: 1 key, never changes)

def validate_key(request_key: str) -> bool:
    return request_key == API_KEY  # Simple string comparison
```

**Características Incluidas:**
- ✅ Validación de key (string comparison)
- ✅ Constante hardcoded (1 key, no DB needed)

**Características NO Incluidas:**
- ❌ Bcrypt hashing (YAGNI: 1 key, local filesystem trusted)
- ❌ Expiration (YAGNI: key never changes)
- ❌ Scopes (YAGNI: 1 user, full access)
- ❌ Revocation (YAGNI: just delete hardcoded key)
- ❌ Multi-tenancy (YAGNI: 1 user)

**Configuración:**
```python
# No config file needed
# Just edit API_KEY constant in code
```

**Performance:**
- Latency: <1ms (in-memory string comparison)
- Memory: 0 bytes (no database, no cache)
- Dependencies: 0 (no bcrypt, no SQLite)

**Upgrade Triggers:**
- If need >5 API keys → Small Business (SQLite table)
- If need multi-user → Small Business (tenant_id field)
- If need expiration → Small Business (expires_at field)

---

### Small Business Profile (60 LOC)

**Contexto del Usuario:**
Small consulting firm (10 employees). Each employee has API key for
their automation scripts. Keys expire after 90 days (security policy).
Need to track who created which key (audit trail).

**Implementation:**
```python
# api_auth.py (Small Business - 60 LOC)
import sqlite3
from datetime import datetime, timedelta

class APIKeyValidator:
    def __init__(self, db_path: str):
        self.conn = sqlite3.connect(db_path)
        self._create_table()

    def _create_table(self):
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS api_keys (
                key_id TEXT PRIMARY KEY,
                key_hash TEXT NOT NULL,
                user_id TEXT NOT NULL,
                expires_at DATETIME NOT NULL,
                created_at DATETIME NOT NULL
            )
        """)

    def validate_key(self, request_key: str) -> bool:
        key_hash = hashlib.sha256(request_key.encode()).hexdigest()
        cursor = self.conn.execute("""
            SELECT expires_at FROM api_keys
            WHERE key_hash = ? AND expires_at > ?
        """, (key_hash, datetime.now()))
        return cursor.fetchone() is not None
```

**Características Incluidas:**
- ✅ Multiple keys (SQLite table)
- ✅ Expiration (expires_at column)
- ✅ Basic hashing (SHA-256, not bcrypt)
- ✅ Audit trail (user_id, created_at)

**Características NO Incluidas:**
- ❌ Bcrypt hashing (SHA-256 sufficient for small scale)
- ❌ Redis caching (SQLite fast enough for <100 req/sec)
- ❌ Scopes (everyone has full access)
- ❌ Key rotation (manual: generate new, delete old)

**Configuración:**
```yaml
api_auth:
  db_path: "./api_keys.db"
  default_expiration_days: 90
```

**Performance:**
- Latency: <10ms (SQLite query)
- Memory: ~1MB (SQLite in-memory cache)
- Dependencies: sqlite3 (built-in Python), hashlib (built-in)

**Upgrade Triggers:**
- If req/sec >100 → Enterprise (Redis cache)
- If need scopes → Enterprise (scopes column)
- If compliance (SOC 2) → Enterprise (bcrypt + audit log)

---

### Enterprise Profile (300 LOC)

**Contexto del Usuario:**
FinTech startup (1000 employees, 10K API clients). Need bcrypt hashing
(bcrypt cost 12), Redis caching (95% hit rate), scopes (read/write
separation), key rotation (quarterly policy), audit trail (HIPAA/SOC 2).

**Implementation:**
```python
# api_auth.py (Enterprise - 300 LOC)
import bcrypt
import redis
from datetime import datetime, timedelta

class APIKeyValidator:
    def __init__(self, db: PostgreSQL, cache: Redis):
        self.db = db
        self.cache = cache
        self.bcrypt_cost = 12

    def validate_key(self, request_key: str) -> AuthContext:
        # 1. Check Redis cache (95% hit rate, <5ms)
        cached = self.cache.get(f"key:{request_key}")
        if cached:
            return AuthContext.from_json(cached)

        # 2. Query PostgreSQL (cache miss, <20ms)
        key_record = self.db.execute("""
            SELECT key_hash, tenant_id, scopes, expires_at
            FROM api_keys
            WHERE key_hash = bcrypt(?, key_hash)
              AND expires_at > NOW()
              AND revoked_at IS NULL
        """, (request_key,))

        if not key_record:
            return None

        # 3. Verify bcrypt (constant-time comparison)
        if not bcrypt.checkpw(request_key.encode(), key_record.key_hash):
            return None

        # 4. Cache result (5-min TTL)
        auth_context = AuthContext(
            tenant_id=key_record.tenant_id,
            scopes=key_record.scopes
        )
        self.cache.setex(
            f"key:{request_key}",
            300,  # 5 minutes
            auth_context.to_json()
        )

        return auth_context
```

**Características Incluidas:**
- ✅ Bcrypt hashing (cost 12, timing attack resistant)
- ✅ Redis caching (95% hit rate, <5ms p95)
- ✅ Scopes (read/write separation)
- ✅ Key rotation (revoke old, generate new)
- ✅ Audit trail (AuditLogger integration)
- ✅ Multi-tenancy (tenant_id isolation)
- ✅ Metrics (Prometheus: validation_latency_ms)

**Características NO Incluidas:**
- ❌ Rate limiting per key (separate RateLimiter primitive)
- ❌ IP whitelisting (separate AccessControl primitive)

**Configuración:**
```yaml
api_auth:
  database:
    url: "postgresql://..."
    pool_size: 20
  cache:
    redis_url: "redis://..."
    ttl_seconds: 300
  bcrypt:
    cost: 12
  key_rotation:
    policy: "quarterly"
    warn_before_days: 30
```

**Performance:**
- Latency: <5ms p95 (cache hit), <20ms p99 (cache miss)
- Memory: ~50MB Redis cache
- Throughput: 10K req/sec (single instance)
- Dependencies: bcrypt, redis, postgresql

**No Further Tiers:**
- Scaling beyond Enterprise is horizontal (more instances, not different architecture)

---

## What Changed from Phase 2 → Phase 5?

| Aspect | Phase 2 (Brief) | Phase 5 (Literate) |
|--------|-----------------|-------------------|
| **Length** | 1 line per profile | ~100 lines per profile |
| **Context** | None | Real user scenario (Darwin, small firm, FinTech) |
| **Code** | No code | Executable examples (copy-paste and run) |
| **Features** | Just LOC count | Explicit included/excluded lists |
| **Configuration** | None | YAML config examples |
| **Performance** | None | Latency, memory, throughput |
| **Upgrade triggers** | None | When to move to next tier |
| **Rationale** | None | Why features included/excluded |

**Key difference:** Phase 5 tells a STORY, not just facts.

---

## Refactoring Pattern

### Step 1: Identify User Scenarios

For each profile, ask:
- **Who is the user?** (Darwin, small firm, enterprise)
- **What problem are they solving?** (automation, multi-user, compliance)
- **What scale?** (transactions/month, users, requests/sec)

### Step 2: Write Context Narrative

```markdown
### {Profile} Profile ({LOC} LOC)

**Contexto del Usuario:**
{2-3 sentences describing WHO uses this profile and WHY}

**Example:**
Darwin tiene una app personal de finanzas. Genera 1 API key para su
script de subida automática. El key nunca expira, no hay multi-tenancy.
```

### Step 3: Show Executable Code

```markdown
**Implementation:**
```python
# Real code (not pseudocode)
# 10-30 lines showing key logic
```
```

### Step 4: Explicit Feature Lists

```markdown
**Características Incluidas:**
- ✅ Feature 1 (why)
- ✅ Feature 2 (why)

**Características NO Incluidas:**
- ❌ Feature 3 (why skipped - YAGNI rationale)
- ❌ Feature 4 (why skipped - wrong tier)
```

### Step 5: Configuration Example

```markdown
**Configuración:**
```yaml
primitive:
  param1: value
  param2: value
```
```

### Step 6: Performance Metrics

```markdown
**Performance:**
- Latency: {p95, p99}
- Memory: {MB}
- Throughput: {req/sec}
- Dependencies: {list}
```

### Step 7: Upgrade Triggers

```markdown
**Upgrade Triggers:**
- If {condition} → {Next tier}
- If {condition} → {Next tier}
```

---

## Primitives to Refactor (33 total)

### Truth Construction (9 primitives)
- [ ] IndexStrategy
- [ ] PaginationEngine
- [ ] ExportEngine
- [ ] ValidationEngine
- [ ] NormalizationLog
- [ ] ParserRegistry
- [ ] ParserVersionManager
- [ ] TransactionQuery
- [ ] ParserSelector

### Audit System (5 primitives)
- [ ] ProvenanceLedger
- [ ] AuditLog
- [ ] TimelineReconstructor
- [ ] RetroactiveCorrector
- [ ] BitemporalQuery

### API/Auth System (8 primitives)
- [ ] DashboardEngine
- [ ] APIKeyValidator
- [ ] APIRouter
- [ ] OAuth2Provider
- [ ] AccessControl
- [ ] APIGateway
- [ ] SchemaRegistry
- [ ] MetricsCollector

### Other OL Primitives (11 primitives)
- [ ] StorageEngine
- [ ] HashCalculator
- [ ] FileArtifact
- [ ] UploadRecord
- [ ] ObservationStore
- [ ] CanonicalStore
- [ ] FuzzyMatcher
- [ ] ClusteringEngine
- [ ] RelationshipStore
- [ ] AccountStore
- [ ] CounterpartyStore

---

## Estimated Effort

| Task | Est. Time | Status |
|------|-----------|--------|
| **Create refactoring pattern** | 2 hours | ✅ Done (this doc) |
| **Refactor 1 primitive (example)** | 3 hours | ⏳ Pending |
| **Refactor remaining 32 primitives** | 96 hours (3 hrs × 32) | ⏳ Pending |
| **Review and consistency pass** | 8 hours | ⏳ Pending |
| **Total** | **109 hours** (~14 days full-time) | 2% complete |

---

## Next Steps

1. ✅ **Create this plan** (Phase 5 roadmap)
2. ⏳ **Refactor 1 primitive as example** (APIKeyValidator - use literate pattern)
3. ⏳ **Get user approval** (pattern looks good?)
4. ⏳ **Batch refactor remaining 32 primitives** (systematic application)
5. ⏳ **Consistency review** (ensure all follow same pattern)
6. ⏳ **Update docs/README.md** (mark Phase 5 complete)

---

## Success Criteria

Phase 5 is complete when ALL of:
- ✅ All 33 primitives have expanded Simplicity Profiles
- ✅ Each profile includes: Context, Code, Features, Config, Performance, Upgrade triggers
- ✅ Profiles follow literate programming style (narrative before code)
- ✅ Examples are executable (real Python/SQL, not pseudocode)
- ✅ Spanish narrative + English code (bilingual pattern)
- ✅ Consistency across all primitives (same structure)

---

**Current Status:** Phase 5 pattern defined, ready for implementation.
**Blocker:** None (pattern documented, can proceed with refactoring).
**Next Action:** Refactor APIKeyValidator as proof-of-concept, then batch-apply to remaining 32.
