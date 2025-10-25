# APIKeyValidator (OL Primitive)

**Domain:** Cross-Cutting (API Management)
**Layer:** Objective Layer (OL)
**Vertical:** 5.5 Public API Contracts
**Created:** 2025-10-25

---

## Purpose

Generates cryptographically secure API keys, validates API keys (bcrypt hash comparison), checks tenant scoping/expiration/revocation, and logs authentication attempts for audit trail.

---

## Responsibilities

1. **Key Generation:** Generate API keys with format `sk_{env}_{random}` (e.g., `tc_live_abc123xyz`)
2. **Key Hashing:** Hash keys with bcrypt before storing (NEVER store plaintext)
3. **Key Validation:** Validate API keys on each request (constant-time comparison via bcrypt)
4. **Tenant Scoping:** Ensure API key can only access scoped tenant_id
5. **Expiration Check:** Reject expired keys (expires_at < now)
6. **Revocation Check:** Reject revoked keys (revoked_at != null)
7. **Usage Tracking:** Update last_used_at timestamp (async, doesn't block request)

---

## Interface

```python
class APIKeyValidator:
    def __init__(self, db: Database, cache: Cache):
        self.db = db
        self.cache = cache  # Redis for caching validated keys

    def generate_key(self, tenant_id: str, scopes: list[str], expires_at: datetime = None, metadata: dict = None) -> str:
        """
        Generate new API key with cryptographically secure random bytes.

        Steps:
        1. Generate 32 random bytes (256 bits entropy)
        2. Base64 encode
        3. Format: sk_{env}_{b64_random} (e.g., tc_live_abc123xyz)
        4. Hash with bcrypt (cost factor 12)
        5. Store hash in database (NEVER store plaintext)
        6. Return plaintext key ONCE (user must save it)

        Returns: API key (plaintext, show once)
        """
        pass

    def validate_key(self, api_key: str) -> Optional[AuthContext]:
        """
        Validate API key and return AuthContext if valid.

        Steps:
        1. Check cache (Redis TTL 5 minutes)
        2. If cache miss: Query database by bcrypt hash
        3. Check expiration (expires_at < now?)
        4. Check revocation (revoked_at != null?)
        5. Update last_used_at (async, don't block)
        6. Cache result (5 minute TTL)
        7. Return AuthContext (tenant_id, scopes, api_key_id)

        Returns None if key is invalid, expired, or revoked.
        """
        pass

    def revoke_key(self, api_key_id: str, revoked_by: str, revocation_reason: str):
        """
        Revoke API key (soft delete for audit trail).

        Steps:
        1. Update api_keys SET revoked_at = NOW(), revoked_by = %s, ...
        2. Invalidate cache (remove from Redis)
        3. Log revocation event (AuditLogger)

        Revoked keys are rejected by validate_key().
        """
        pass

    def rotate_key(self, api_key_id: str, tenant_id: str, scopes: list[str]) -> str:
        """
        Rotate API key (revoke old, generate new).

        Steps:
        1. Revoke old key (soft delete)
        2. Generate new key (same tenant_id, scopes)
        3. Return new key

        Use case: Key leaked, user wants to rotate without downtime.
        """
        pass

    def hash_key(self, api_key: str) -> str:
        """
        Hash API key with bcrypt (cost factor 12).

        Uses bcrypt for constant-time comparison (prevent timing attacks).
        """
        pass

    def check_cache(self, api_key: str) -> Optional[AuthContext]:
        """
        Check Redis cache for validated key.

        Cache key: api_key:<hash>
        Cache value: JSON-serialized AuthContext
        TTL: 5 minutes

        Returns None if cache miss.
        """
        pass

    def cache_result(self, api_key: str, auth_context: AuthContext):
        """
        Cache validated key in Redis (TTL 5 minutes).

        Reduces database load (validation happens on every request).
        """
        pass
```

---

## Data Model

**APIKey (Database Table):**

```sql
CREATE TABLE api_keys (
  api_key_id UUID PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  key_hash VARCHAR(255) NOT NULL,  -- bcrypt hash (NEVER plaintext)
  key_prefix VARCHAR(16) NOT NULL,  -- e.g., "tc_live_abc123" (for UI display)
  scopes TEXT[] NOT NULL,  -- ['read', 'write', 'admin']
  expires_at TIMESTAMP NULL,  -- NULL = never expires
  last_used_at TIMESTAMP NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  revoked_at TIMESTAMP NULL,
  revoked_by VARCHAR(255) NULL,
  revocation_reason TEXT NULL,
  metadata JSONB  -- { "name": "Production Key", "created_by": "usr_admin" }
);

CREATE INDEX idx_api_keys_tenant ON api_keys(tenant_id);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
```

**AuthContext (In-Memory):**

```python
@dataclass
class AuthContext:
    tenant_id: str
    user_id: Optional[str]  # None for API keys
    scopes: list[str]  # ['read', 'write', 'admin']
    api_key_id: str
```

---

## Implementation Notes

### Key Format

```
sk_{environment}_{random_base64}

Examples:
- tc_live_3K7mP9xQ2jR8vN5wL1tY4uA6bC0dE  (Production key)
- tc_test_8uA6bC0dE3K7mP9xQ2jR8vN5wL1tY  (Test/staging key)
- tc_sandbox_1tY4uA6bC0dE3K7mP9xQ2jR8vN5w  (Sandbox key, reset daily)
```

**Why this format?**
- **Prefix `sk_`:** Easy to detect in secret scanning tools (GitHub, GitGuardian)
- **Environment tag:** Prevents accidental use of test keys in production
- **Random base64:** 32 bytes = 256 bits entropy (cryptographically secure)

### bcrypt Hashing

```python
import bcrypt

# Generate key
key_bytes = secrets.token_bytes(32)  # 256 bits entropy
key_b64 = base64.urlsafe_b64encode(key_bytes).decode('utf-8')
api_key = f"tc_live_{key_b64}"

# Hash for storage (NEVER store plaintext)
key_hash = bcrypt.hashpw(api_key.encode('utf-8'), bcrypt.gensalt(rounds=12))

# Store hash
db.execute("INSERT INTO api_keys (api_key_id, tenant_id, key_hash, ...) VALUES (%s, %s, %s, ...)",
           (uuid4(), tenant_id, key_hash))

# Validate (constant-time comparison)
result = db.query("SELECT * FROM api_keys WHERE key_hash = crypt(%s, key_hash)", (api_key,))
# bcrypt's crypt() function does constant-time comparison internally
```

### Redis Caching Strategy

```python
# Cache validated key (5 minute TTL)
cache_key = f"api_key:{api_key[:16]}"  # Use prefix to avoid storing full key in Redis
cache_value = json.dumps({
    'tenant_id': 'tenant_acme',
    'scopes': ['read', 'write'],
    'api_key_id': 'key_abc123'
})
redis.setex(cache_key, 300, cache_value)  # TTL 300 seconds (5 minutes)

# Check cache on next request
cached = redis.get(cache_key)
if cached:
    auth_context = AuthContext(**json.loads(cached))
    # Skip database query, return cached result
else:
    # Cache miss, query database
    pass
```

**Why 5-minute TTL?**
- Balances database load reduction vs staleness risk
- Revoked keys are rejected within 5 minutes (acceptable for most use cases)
- For higher security, reduce TTL to 1 minute OR invalidate cache on revocation

---

## Example Usage

```python
# Example 1: Generate API key
validator = APIKeyValidator(db, cache)
api_key = validator.generate_key(
    tenant_id="tenant_acme",
    scopes=["read", "write"],
    expires_at=None,  # Never expires
    metadata={"name": "Production Key", "created_by": "usr_admin"}
)
# Returns: "tc_live_3K7mP9xQ2jR8vN5wL1tY4uA6bC0dE"
# (Show to user ONCE, they must save it)

# Example 2: Validate API key (success)
api_key = "tc_live_3K7mP9xQ2jR8vN5wL1tY4uA6bC0dE"
auth_context = validator.validate_key(api_key)
# Returns: AuthContext(tenant_id="tenant_acme", scopes=["read", "write"], api_key_id="key_abc123")

# Example 3: Validate API key (expired)
api_key = "tc_live_expired_key"
auth_context = validator.validate_key(api_key)
# Returns: None (key expired)

# Example 4: Validate API key (revoked)
api_key = "tc_live_revoked_key"
auth_context = validator.validate_key(api_key)
# Returns: None (key revoked)

# Example 5: Revoke API key
validator.revoke_key(
    api_key_id="key_abc123",
    revoked_by="usr_admin",
    revocation_reason="Key leaked on GitHub"
)
# Sets revoked_at = NOW(), invalidates cache

# Example 6: Rotate API key
old_key_id = "key_abc123"
new_api_key = validator.rotate_key(old_key_id, "tenant_acme", ["read", "write"])
# Returns: "tc_live_9xQ2jR8vN5wL1tY4uA6bC0dE3K7mP" (new key)
# Old key is revoked, new key has same scopes
```

---

## Domain Applicability

**Universal Across All Domains:**

- **Finance:** Bank generates API key for automated statement uploads
- **Healthcare:** Hospital generates API key for EHR integration
- **Legal:** Law firm generates API key for contract extraction
- **Research:** University generates API key for paper processing
- **E-Commerce:** SaaS platform generates API key for invoice reconciliation
- **HR SaaS:** HR platform generates API key for W-2 processing
- **Insurance:** Insurance company generates API key for claim processing

**Pattern:** APIKeyValidator is domain-agnostic, validates any API key regardless of use case.

---

## Related Primitives

- **APIGateway (OL):** Uses APIKeyValidator to authenticate requests
- **OAuth2Provider (OL):** Alternative authentication method (for user-delegated access)
- **AuditLogger (OL, from 5.4):** Logs key generation/revocation events
- **TenantIsolator (OL, from 5.4):** Ensures API key can't access other tenants' data

---

## Testing

```python
# Unit Test: Key generation format
def test_generate_key_format():
    validator = APIKeyValidator(db, cache)
    key = validator.generate_key('tenant_acme', ['read'])
    assert key.startswith('tc_test_')  # Test environment
    assert len(key) > 40  # Sufficient entropy

# Unit Test: Key validation success
def test_validate_key_success():
    validator = APIKeyValidator(db, cache)
    # Generate key
    api_key = validator.generate_key('tenant_acme', ['read', 'write'])
    # Validate key
    auth_context = validator.validate_key(api_key)
    assert auth_context.tenant_id == 'tenant_acme'
    assert 'read' in auth_context.scopes

# Unit Test: Key validation expired
def test_validate_key_expired():
    validator = APIKeyValidator(db, cache)
    # Create expired key
    expired_key = create_expired_key('tenant_acme', expires_at=datetime(2020, 1, 1))
    # Validate key
    auth_context = validator.validate_key(expired_key)
    assert auth_context is None  # Expired

# Unit Test: Key revocation
def test_revoke_key():
    validator = APIKeyValidator(db, cache)
    # Generate key
    api_key = validator.generate_key('tenant_acme', ['read'])
    # Revoke key
    validator.revoke_key(api_key_id, 'usr_admin', 'Key leaked')
    # Validate key (should fail)
    auth_context = validator.validate_key(api_key)
    assert auth_context is None  # Revoked

# Unit Test: Cache hit
def test_cache_hit():
    validator = APIKeyValidator(db, cache)
    # Generate key
    api_key = validator.generate_key('tenant_acme', ['read'])
    # First validation (cache miss, queries DB)
    auth_context1 = validator.validate_key(api_key)
    # Second validation (cache hit, no DB query)
    auth_context2 = validator.validate_key(api_key)
    assert auth_context1.tenant_id == auth_context2.tenant_id
    # Verify no second DB query (mock/spy on db.query)

# Integration Test: Key rotation
def test_key_rotation():
    validator = APIKeyValidator(db, cache)
    # Generate key
    old_key = validator.generate_key('tenant_acme', ['read', 'write'])
    old_key_id = db.query("SELECT api_key_id FROM api_keys WHERE key_prefix = %s", (old_key[:16],))['api_key_id']
    # Rotate key
    new_key = validator.rotate_key(old_key_id, 'tenant_acme', ['read', 'write'])
    # Old key should be revoked
    assert validator.validate_key(old_key) is None
    # New key should work
    assert validator.validate_key(new_key).tenant_id == 'tenant_acme'
```

---

## Security Considerations

1. **Never Store Plaintext Keys:**
   - Hash with bcrypt (cost factor 12) before storing
   - Database breach does NOT expose keys

2. **Constant-Time Comparison:**
   - bcrypt's `crypt()` prevents timing attacks
   - Attacker can't guess keys by measuring comparison time

3. **Key Rotation:**
   - Provide easy key rotation (revoke old, generate new)
   - Encourage 12-month rotation policy

4. **Revocation:**
   - Soft delete (set revoked_at, keep record for audit)
   - Cache invalidation ensures revoked keys rejected within 5 minutes

5. **Expiration:**
   - Support time-limited keys (expires_at)
   - Use for temporary contractor access

6. **Audit Trail:**
   - Log all key generation/revocation events
   - Include revocation_reason for compliance

---

## Performance

- **Key Generation:** p95 < 50ms (bcrypt hashing with cost 12)
- **Key Validation (Cache Hit):** p95 < 5ms (Redis lookup)
- **Key Validation (Cache Miss):** p95 < 20ms (bcrypt comparison + DB query)
- **Cache Hit Rate:** ~95% (most requests use cached result)

**Optimization:** Redis cache reduces DB load by 95%, critical for high-traffic APIs.

---

## Metadata

- **Lines of Code:** ~300 (Python implementation)
- **Dependencies:** Database, Redis, bcrypt, secrets module
- **Deployment:** Used by APIGateway as middleware component
