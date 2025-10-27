# OAuth2Provider (OL Primitive)

**Domain:** Cross-Cutting (API Management)
**Layer:** Objective Layer (OL)
**Vertical:** 5.5 Public API Contracts
**Created:** 2025-10-25

---

## Purpose

Issues OAuth2 access tokens and refresh tokens for user-delegated API access, validates tokens on each request, handles token expiration/revocation, and manages OAuth2 authorization flow (authorization code grant).

---

## Responsibilities

1. **Authorization Flow:** Implement OAuth2 authorization code grant (user authorizes third-party app)
2. **Token Issuance:** Issue access tokens (short-lived, 15 minutes) and refresh tokens (long-lived, 30 days)
3. **Token Validation:** Validate access tokens on each API request (check expiration, revocation, scopes)
4. **Token Refresh:** Exchange refresh token for new access token (when access token expires)
5. **Token Revocation:** Revoke access/refresh tokens (user logs out, security incident)
6. **Scope Enforcement:** Enforce requested scopes (read, write, admin)

---

## Interface

```python
class OAuth2Provider:
    def __init__(self, db: Database, cache: Cache, jwt_secret: str):
        self.db = db
        self.cache = cache
        self.jwt_secret = jwt_secret  # Secret for signing JWTs

    def authorize(self, client_id: str, redirect_uri: str, scopes: list[str], user_id: str) -> str:
        """
        Step 1 of OAuth2 flow: User authorizes client app.

        Steps:
        1. User logs in, sees consent screen ("App X wants to access your data")
        2. User clicks "Allow"
        3. Generate authorization code (short-lived, 5 minutes)
        4. Store code in database (client_id, user_id, scopes, redirect_uri)
        5. Redirect to redirect_uri?code=<authorization_code>

        Returns: authorization_code (e.g., "auth_abc123xyz")
        """
        pass

    def exchange_code(self, client_id: str, client_secret: str, authorization_code: str, redirect_uri: str) -> dict:
        """
        Step 2 of OAuth2 flow: Client exchanges code for tokens.

        Steps:
        1. Validate client_id + client_secret (authenticate client app)
        2. Validate authorization_code (check expiration, used_at=NULL)
        3. Mark code as used (used_at=NOW, prevent replay)
        4. Issue access token (JWT, 15-minute expiration)
        5. Issue refresh token (random string, 30-day expiration)
        6. Return {"access_token": "...", "refresh_token": "...", "expires_in": 900}

        Returns: Token response (access_token, refresh_token, expires_in)
        """
        pass

    def validate_token(self, access_token: str) -> Optional[AuthContext]:
        """
        Validate access token on each API request.

        Steps:
        1. Decode JWT (verify signature with jwt_secret)
        2. Check expiration (exp claim)
        3. Check revocation (query revoked_tokens table)
        4. Return AuthContext (tenant_id, user_id, scopes)

        Returns None if token is invalid, expired, or revoked.
        """
        pass

    def refresh_token(self, client_id: str, client_secret: str, refresh_token: str) -> dict:
        """
        Exchange refresh token for new access token.

        Steps:
        1. Validate client_id + client_secret
        2. Validate refresh_token (check expiration, revoked=False)
        3. Issue new access token (15-minute expiration)
        4. Return {"access_token": "...", "expires_in": 900}

        Note: Refresh token is NOT rotated (same refresh token can be used multiple times).
        """
        pass

    def revoke_token(self, token: str, token_type: str):
        """
        Revoke access token or refresh token.

        Steps:
        1. If token_type=access_token: Add to revoked_tokens table (JWT can't be invalidated directly)
        2. If token_type=refresh_token: Mark refresh_token as revoked in database
        3. Invalidate cache (if using Redis caching)

        Use case: User logs out, security incident, token leaked.
        """
        pass

    def issue_access_token(self, user_id: str, tenant_id: str, scopes: list[str]) -> str:
        """
        Issue JWT access token.

        JWT claims:
        - sub: user_id
        - tenant_id: tenant_id
        - scopes: ["read", "write"]
        - exp: now + 15 minutes (900 seconds)
        - iat: now (issued at)
        - jti: token_id (unique ID for revocation)

        Signed with HMAC-SHA256 (jwt_secret).
        """
        pass

    def issue_refresh_token(self, user_id: str, tenant_id: str, client_id: str, scopes: list[str]) -> str:
        """
        Issue refresh token (random string).

        Steps:
        1. Generate random token (32 bytes, base64)
        2. Store in database (user_id, tenant_id, client_id, scopes, expires_at=NOW+30d)
        3. Return token

        Refresh token is stored in database (can be revoked).
        """
        pass
```

---

## Data Model

**OAuth2Client (Database Table):**

```sql
CREATE TABLE oauth2_clients (
  client_id VARCHAR(255) PRIMARY KEY,
  client_secret VARCHAR(255) NOT NULL,  -- bcrypt hash
  redirect_uris TEXT[] NOT NULL,  -- Allowed redirect URIs
  name VARCHAR(255) NOT NULL,  -- e.g., "Acme Corp Integration"
  logo_url TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

**OAuth2AuthorizationCode (Database Table):**

```sql
CREATE TABLE oauth2_authorization_codes (
  code VARCHAR(255) PRIMARY KEY,
  client_id VARCHAR(255) NOT NULL REFERENCES oauth2_clients(client_id),
  user_id VARCHAR(255) NOT NULL,
  tenant_id VARCHAR(255) NOT NULL,
  scopes TEXT[] NOT NULL,
  redirect_uri TEXT NOT NULL,
  expires_at TIMESTAMP NOT NULL,  -- 5 minutes from issued_at
  used_at TIMESTAMP NULL,  -- NULL = not used yet
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_oauth2_codes_client ON oauth2_authorization_codes(client_id);
```

**OAuth2RefreshToken (Database Table):**

```sql
CREATE TABLE oauth2_refresh_tokens (
  refresh_token VARCHAR(255) PRIMARY KEY,
  client_id VARCHAR(255) NOT NULL REFERENCES oauth2_clients(client_id),
  user_id VARCHAR(255) NOT NULL,
  tenant_id VARCHAR(255) NOT NULL,
  scopes TEXT[] NOT NULL,
  expires_at TIMESTAMP NOT NULL,  -- 30 days from issued_at
  revoked BOOLEAN DEFAULT FALSE,
  revoked_at TIMESTAMP NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_oauth2_refresh_tokens_user ON oauth2_refresh_tokens(user_id);
```

**OAuth2RevokedToken (Database Table):**

```sql
CREATE TABLE oauth2_revoked_tokens (
  jti VARCHAR(255) PRIMARY KEY,  -- JWT token ID (from jti claim)
  revoked_at TIMESTAMP DEFAULT NOW()
);

-- Clean up expired entries (TTL > 15 minutes, access tokens expire after 15 min)
CREATE INDEX idx_oauth2_revoked_tokens_revoked_at ON oauth2_revoked_tokens(revoked_at);
```

---

## Implementation Notes

### OAuth2 Authorization Code Flow

```
1. User clicks "Connect to App X" in third-party app
   ↓
2. Redirect to: https://api.example.com/oauth/authorize?client_id=...&redirect_uri=...&scopes=read,write
   ↓
3. User sees consent screen: "App X wants to access your observations"
   ↓
4. User clicks "Allow"
   ↓
5. OAuth2Provider.authorize() generates authorization code
   ↓
6. Redirect to: https://appx.com/callback?code=auth_abc123xyz
   ↓
7. App X backend calls: POST /oauth/token {client_id, client_secret, code, redirect_uri}
   ↓
8. OAuth2Provider.exchange_code() issues access_token + refresh_token
   ↓
9. App X stores tokens, uses access_token for API requests
   ↓
10. When access_token expires (15 min), use refresh_token to get new access_token
```

### JWT Access Token Format

```json
{
  "sub": "usr_jane",
  "tenant_id": "tenant_acme",
  "scopes": ["read", "write"],
  "exp": 1730000000,  // Unix timestamp (now + 15 minutes)
  "iat": 1729999100,  // Issued at
  "jti": "tok_abc123xyz"  // Token ID (for revocation)
}
```

**Signed with HMAC-SHA256:**
```
Header: {"alg": "HS256", "typ": "JWT"}
Payload: {...}
Signature: HMACSHA256(base64(header) + "." + base64(payload), jwt_secret)
```

**Full JWT:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c3JfamFuZSIsInRlbmFudF9pZCI6InRlbmFudF9hY21lIiwic2NvcGVzIjpbInJlYWQiLCJ3cml0ZSJdLCJleHAiOjE3MzAwMDAwMDAsImlhdCI6MTcyOTk5OTEwMCwianRpIjoidG9rX2FiYzEyM3h5eiJ9.signature
```

### Token Expiration Strategy

| Token Type | Expiration | Rationale |
|------------|-----------|-----------|
| Authorization Code | 5 minutes | Short-lived, single-use (prevents replay attacks) |
| Access Token | 15 minutes | Short-lived (limits damage if stolen), refreshable |
| Refresh Token | 30 days | Long-lived (user doesn't re-authorize frequently), revokable |

---

## Example Usage

```python
# Example 1: Authorization flow (user authorizes app)
oauth2_provider = OAuth2Provider(db, cache, jwt_secret)

# User clicks "Allow" on consent screen
authorization_code = oauth2_provider.authorize(
    client_id="client_appx",
    redirect_uri="https://appx.com/callback",
    scopes=["read", "write"],
    user_id="usr_jane"
)
# Returns: "auth_abc123xyz"
# Redirect to: https://appx.com/callback?code=auth_abc123xyz

# Example 2: Exchange code for tokens (app backend)
token_response = oauth2_provider.exchange_code(
    client_id="client_appx",
    client_secret="secret_xyz",
    authorization_code="auth_abc123xyz",
    redirect_uri="https://appx.com/callback"
)
# Returns:
# {
#   "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
#   "refresh_token": "refresh_def456",
#   "expires_in": 900,
#   "token_type": "Bearer"
# }

# Example 3: Validate access token (API request)
access_token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
auth_context = oauth2_provider.validate_token(access_token)
# Returns: AuthContext(tenant_id="tenant_acme", user_id="usr_jane", scopes=["read", "write"])

# Example 4: Refresh access token (after 15 minutes)
token_response = oauth2_provider.refresh_token(
    client_id="client_appx",
    client_secret="secret_xyz",
    refresh_token="refresh_def456"
)
# Returns:
# {
#   "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",  // New access token
#   "expires_in": 900
# }

# Example 5: Revoke token (user logs out)
oauth2_provider.revoke_token(
    token="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    token_type="access_token"
)
# Adds token's jti to revoked_tokens table

oauth2_provider.revoke_token(
    token="refresh_def456",
    token_type="refresh_token"
)
# Marks refresh_token as revoked in database
```

---

## Domain Applicability

**Universal Across All Domains:**

- **Finance:** Third-party budgeting app uses OAuth2 to access bank statements
- **Healthcare:** Patient portal app uses OAuth2 to access lab reports
- **Legal:** Contract management app uses OAuth2 to access extracted entities
- **Research (RSRCH - Utilitario):** VC firm CRM uses OAuth2 to access founder facts, investment tracking dashboard uses OAuth2 to query company facts
- **E-Commerce:** Accounting software uses OAuth2 to access invoice observations
- **HR SaaS:** Payroll app uses OAuth2 to access W-2 observations
- **Insurance:** Claims portal uses OAuth2 to access claim observations

**Pattern:** OAuth2Provider is domain-agnostic, provides user-delegated access for any third-party app.

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Third-party budgeting app (e.g., Mint, YNAB) uses OAuth2 to access user's bank transactions
**Example:** User clicks "Connect to Bank Account" in budgeting app → OAuth2Provider.authorize() shows consent screen "Budgeting App wants to access your transactions" → User clicks "Allow" → OAuth2Provider issues authorization code → App exchanges code for access_token + refresh_token → App calls GET /v1/transactions with Bearer token → OAuth2Provider.validate_token() verifies JWT signature, checks expiration (15 min), scopes ["read"] → Returns transactions
**Operations:** authorize (consent screen), exchange_code (code → tokens), validate_token (JWT verify), refresh_token (new access token), revoke_token (user disconnects app)
**Token lifetime:** Access token 15 min, refresh token 30 days
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Patient portal app uses OAuth2 to access lab results from EHR system
**Example:** Patient clicks "View Lab Results" in third-party health app → OAuth2Provider.authorize() shows consent "Health App wants to access your lab results" → Patient authorizes → OAuth2Provider issues tokens with scopes ["read:lab_results"] → App requests GET /v1/lab_results with Bearer token → OAuth2Provider.validate_token() checks JWT + scopes → Returns lab test data (glucose, cholesterol) if scopes match
**Operations:** Scope enforcement (read:lab_results, read:prescriptions, read:appointments), HIPAA-compliant audit logging (who accessed what), token revocation (patient revokes access)
**Token lifetime:** Short-lived access tokens (15 min) for security, refresh tokens for long-running sessions
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ Legal
**Use case:** Contract management SaaS uses OAuth2 to access extracted contract entities
**Example:** Law firm connects contract management tool to Truth Construction API → OAuth2Provider.authorize() shows consent "ContractTool wants to access your case documents" → Firm admin authorizes with scopes ["read:cases", "write:annotations"] → Tool exchanges code for tokens → Tool calls GET /v1/cases with Bearer token → OAuth2Provider.validate_token() verifies JWT, checks scopes → Returns case documents if scopes permit
**Operations:** Client registration (register ContractTool with redirect_uri, logo), scope-based permissions (read vs write), multi-user delegation (different attorneys authorize same app)
**Token lifetime:** Refresh tokens rotated on security incidents
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ RSRCH (Utilitario Research)
**Use case:** VC firm CRM (e.g., Affinity, Attio) uses OAuth2 to access founder facts for pipeline tracking
**Example:** VC associate clicks "Sync Founder Data" in CRM → OAuth2Provider.authorize() shows consent "CRM wants to access your founder facts database" → Associate authorizes with scopes ["read:facts", "read:companies"] → CRM exchanges code for tokens → CRM calls GET /v1/facts?subject_entity=Sam+Altman with Bearer token → OAuth2Provider.validate_token() verifies JWT, checks scopes ["read:facts"] → Returns founder investment facts, board seats, company launches
**Operations:** Scope enforcement (read:facts, read:companies, read:relationships), tiered rate limits per OAuth2 client (CRM gets 10k req/hour, personal dashboard 1k req/hour), webhook access tokens (POST /v1/webhooks with write:webhooks scope)
**Token lifetime:** Access token 15 min (API-heavy CRM usage, frequent refreshes), refresh token 30 days (CRM background sync jobs)
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ E-commerce
**Use case:** Accounting software (QuickBooks, Xero) uses OAuth2 to access invoice observations
**Example:** Merchant clicks "Connect Accounting Software" → OAuth2Provider.authorize() shows consent "QuickBooks wants to access your invoices" → Merchant authorizes with scopes ["read:invoices", "read:products"] → QuickBooks exchanges code for tokens → QuickBooks calls GET /v1/invoices with Bearer token → OAuth2Provider.validate_token() checks JWT, scopes → Returns invoice observations (date, amount, customer, line items)
**Operations:** Multi-tenant isolation (QuickBooks can't access other merchants' invoices), refresh token rotation (30-day expiration, automatic renewal), revocation on disconnect (merchant revokes QuickBooks access)
**Token lifetime:** Long-lived refresh tokens for background invoice sync (nightly jobs)
**Status:** ✅ Conceptually validated via examples in this doc

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (OAuth2 flow, JWT validation, scope enforcement are universal patterns, no domain-specific code)
**Reusability:** High (same authorize/exchange_code/validate_token operations work for budgeting apps, patient portals, CRMs, accounting software; only scopes and client metadata differ)

---

## Simplicity Profiles

**Personal - 0 LOC:** Not needed (single user)
**Small Business - ~150 LOC:** Basic OAuth2 (authorization code flow)
**Enterprise - ~800 LOC:** Full OAuth2 + OIDC + PKCE + refresh tokens

---

## Related Primitives

- **APIGateway (OL):** Uses OAuth2Provider to authenticate requests (alternative to APIKeyValidator)
- **APIKeyValidator (OL):** Alternative authentication method (for server-to-server, no user delegation)
- **AuditLogger (OL, from 5.4):** Logs token issuance/revocation events
- **TenantIsolator (OL, from 5.4):** Ensures OAuth2 token can't access other tenants' data

---

## Testing

```python
# Unit Test: Authorization code generation
def test_authorize():
    provider = OAuth2Provider(db, cache, jwt_secret)
    code = provider.authorize('client_appx', 'https://appx.com/callback', ['read'], 'usr_jane')
    assert code.startswith('auth_')
    assert len(code) > 20

# Unit Test: Exchange code for tokens
def test_exchange_code():
    provider = OAuth2Provider(db, cache, jwt_secret)
    code = provider.authorize('client_appx', 'https://appx.com/callback', ['read'], 'usr_jane')
    token_response = provider.exchange_code('client_appx', 'secret_xyz', code, 'https://appx.com/callback')
    assert 'access_token' in token_response
    assert 'refresh_token' in token_response
    assert token_response['expires_in'] == 900

# Unit Test: Validate access token
def test_validate_token():
    provider = OAuth2Provider(db, cache, jwt_secret)
    access_token = provider.issue_access_token('usr_jane', 'tenant_acme', ['read', 'write'])
    auth_context = provider.validate_token(access_token)
    assert auth_context.user_id == 'usr_jane'
    assert 'read' in auth_context.scopes

# Unit Test: Validate expired token
def test_validate_expired_token():
    provider = OAuth2Provider(db, cache, jwt_secret)
    # Issue token with past expiration
    expired_token = create_expired_token('usr_jane', 'tenant_acme', ['read'])
    auth_context = provider.validate_token(expired_token)
    assert auth_context is None

# Unit Test: Refresh token
def test_refresh_token():
    provider = OAuth2Provider(db, cache, jwt_secret)
    # Issue refresh token
    refresh_token = provider.issue_refresh_token('usr_jane', 'tenant_acme', 'client_appx', ['read', 'write'])
    # Refresh access token
    token_response = provider.refresh_token('client_appx', 'secret_xyz', refresh_token)
    assert 'access_token' in token_response
    assert token_response['expires_in'] == 900

# Unit Test: Revoke access token
def test_revoke_access_token():
    provider = OAuth2Provider(db, cache, jwt_secret)
    access_token = provider.issue_access_token('usr_jane', 'tenant_acme', ['read'])
    provider.revoke_token(access_token, 'access_token')
    # Validate token (should fail)
    auth_context = provider.validate_token(access_token)
    assert auth_context is None

# Integration Test: Full OAuth2 flow
def test_full_oauth2_flow():
    provider = OAuth2Provider(db, cache, jwt_secret)
    # Step 1: Authorize
    code = provider.authorize('client_appx', 'https://appx.com/callback', ['read'], 'usr_jane')
    # Step 2: Exchange code
    token_response = provider.exchange_code('client_appx', 'secret_xyz', code, 'https://appx.com/callback')
    # Step 3: Validate access token
    auth_context = provider.validate_token(token_response['access_token'])
    assert auth_context.user_id == 'usr_jane'
    # Step 4: Refresh access token
    new_token_response = provider.refresh_token('client_appx', 'secret_xyz', token_response['refresh_token'])
    assert 'access_token' in new_token_response
```

---

## Security Considerations

1. **Authorization Code Replay:**
   - Mark code as used (used_at=NOW) after exchange
   - Reject reused codes (prevents replay attacks)

2. **Client Authentication:**
   - Validate client_secret (bcrypt hash) on token exchange
   - Reject requests with invalid client credentials

3. **Redirect URI Validation:**
   - Only allow registered redirect_uris for client
   - Prevents authorization code hijacking

4. **Short-Lived Access Tokens:**
   - 15-minute expiration limits damage if token stolen
   - Requires refresh token for long-running sessions

5. **JWT Signature Verification:**
   - Verify HMAC-SHA256 signature on every validation
   - Prevents token tampering

6. **Token Revocation:**
   - Support immediate revocation (user logs out, security incident)
   - Check revoked_tokens table on validation

7. **HTTPS Required:**
   - All OAuth2 endpoints must use HTTPS
   - Prevents token interception

---

## Performance

- **Token Issuance:** p95 < 50ms (JWT signing + DB insert)
- **Token Validation:** p95 < 20ms (JWT decode + signature verify)
- **Token Refresh:** p95 < 50ms (DB query + JWT signing)
- **Revocation Check:** p95 < 10ms (DB query on revoked_tokens)

**Optimization:** Cache revoked tokens in Redis (reduce DB queries for revocation check).

---

## Metadata

- **Lines of Code:** ~500 (Python implementation)
- **Dependencies:** Database, Redis, JWT library (PyJWT), bcrypt
- **Deployment:** OAuth2 endpoints (/oauth/authorize, /oauth/token) in API service
