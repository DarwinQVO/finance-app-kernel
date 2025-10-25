# ADR-0041: API Key vs OAuth2 - When to Use Which

**Status:** ✅ Accepted
**Date:** 2025-10-25
**Vertical:** 5.5 Public API Contracts
**Deciders:** Architecture Team, Security Team, Developer Experience Team

---

## Context

Our public API supports two authentication methods:

1. **API Keys:** Long-lived tokens (tc_live_abc123...) scoped to tenant, managed by tenant admin
2. **OAuth2:** User-delegated access with short-lived access tokens (15 min) + refresh tokens (30 days)

Customers are confused about when to use which method. We need clear guidance based on:

1. **Use case:** Server-to-server vs third-party app vs mobile app
2. **Security:** Access to user's data vs tenant's data
3. **User experience:** Admin-only setup vs end-user authorization
4. **Scope granularity:** Tenant-wide access vs user-specific scopes

**Key Questions:**
- When should customers use API keys? (simple, but less secure)
- When should customers use OAuth2? (complex, but more secure)
- Can we support both simultaneously? (yes, but needs clear guidance)
- What are security implications of each method?

---

## Decision

We will support **both API keys and OAuth2** with **clear use-case-based guidance**.

### Use Case Decision Tree

```
┌─────────────────────────────────────────────────────────────────┐
│  Who is accessing the API?                                      │
└─────────────────────────────────────────────────────────────────┘
                        │
            ┌───────────┴───────────┐
            ▼                       ▼
   ┌──────────────────┐    ┌──────────────────┐
   │  Internal System │    │  Third-Party App │
   │  (owned by       │    │  (external,      │
   │   tenant)        │    │   customer uses) │
   └──────────────────┘    └──────────────────┘
            │                       │
            ▼                       ▼
   ┌──────────────────┐    ┌──────────────────┐
   │  USE API KEYS    │    │  USE OAUTH2      │
   │                  │    │                  │
   │  Examples:       │    │  Examples:       │
   │  - ERP system    │    │  - Mobile app    │
   │  - Cron job      │    │  - Browser ext   │
   │  - ETL pipeline  │    │  - SaaS integr   │
   └──────────────────┘    └──────────────────┘
```

### 1. Use API Keys When:

**Scenario:** Internal system owned by tenant (server-to-server integration)

| Use Case | Example | Why API Key? |
|----------|---------|--------------|
| **Automated workflows** | Cron job uploads bank statements daily | No user interaction, tenant owns system |
| **ERP integration** | SAP system syncs observations to ERP | Server-to-server, tenant controls both systems |
| **ETL pipelines** | Data warehouse pulls observations nightly | Batch processing, no per-user scopes needed |
| **Microservices** | Internal service calls API | Tenant controls both services, simpler auth |
| **CI/CD pipelines** | GitHub Actions uploads test results | Automated, no user authorization flow |

**Characteristics:**
- ✅ **Server-to-server:** Backend system calls API (not browser/mobile)
- ✅ **Tenant-owned:** Tenant controls the client system (not third-party)
- ✅ **Long-lived:** Integration runs indefinitely (not tied to user session)
- ✅ **Simple setup:** Admin generates key once, copy-pastes into config
- ❌ **No user-specific scopes:** API key is tenant-wide (can't scope to single user's data)

**Security Properties:**
- Tenant-scoped (cannot access other tenants' data)
- Scope-based permissions (read, write, admin)
- Can expire (set expires_at for temporary access)
- Can be revoked instantly (admin clicks "Revoke" in UI)
- Must be stored securely (environment variable, secret manager, NOT in code)

**Example Flow:**
```bash
# 1. Admin generates API key in UI
API_KEY="tc_live_3K7mP9xQ2jR8vN5wL1tY4uA6bC0dE"

# 2. Admin stores key in environment variable
export TRUTH_API_KEY="tc_live_3K7mP9xQ2jR8vN5wL1tY4uA6bC0dE"

# 3. Cron job uses key to upload
curl -X POST https://api.truthconstruction.com/v1/uploads \
  -H "Authorization: Bearer $TRUTH_API_KEY" \
  -F "file=@statement.pdf"
```

### 2. Use OAuth2 When:

**Scenario:** Third-party app needs user-delegated access

| Use Case | Example | Why OAuth2? |
|----------|---------|-------------|
| **Mobile apps** | User installs iOS app, authorizes access | User grants permission, not tenant admin |
| **Browser extensions** | Chrome extension parses PDFs from Gmail | Runs in user's browser, needs user consent |
| **SaaS integrations** | Zapier/IFTTT connects to user's account | Third-party app, user authorizes their data |
| **Public marketplace** | App Store app integrates with API | User-facing app, not internal system |
| **Multi-tenant SaaS** | Accounting software accesses client data | Each end-user authorizes their own data |

**Characteristics:**
- ✅ **Third-party:** Customer uses app built by someone else (not tenant)
- ✅ **User authorization:** End-user clicks "Allow" consent screen (not admin)
- ✅ **User-specific scopes:** Access limited to authorizing user's data
- ✅ **Short-lived tokens:** Access token expires after 15 minutes (security)
- ✅ **Revocable by user:** User can revoke access anytime (OAuth2 consent screen)
- ❌ **Complex setup:** Requires OAuth2 flow (authorization code, token exchange)

**Security Properties:**
- User-scoped (can only access authorizing user's data, not entire tenant)
- Fine-grained permissions (user chooses which scopes to grant)
- Short-lived access tokens (15 min expiration limits damage if stolen)
- Refresh tokens can be revoked (user logs out, revoke all tokens)
- No shared secrets (user never sees or stores tokens manually)

**Example Flow:**
```
1. User clicks "Connect to Truth Construction" in third-party app
   ↓
2. Redirect to: https://api.truthconstruction.com/oauth/authorize?client_id=...&scopes=read
   ↓
3. User sees consent screen: "App X wants to access your observations"
   ↓
4. User clicks "Allow"
   ↓
5. Redirect to: https://appx.com/callback?code=auth_abc123
   ↓
6. App X exchanges code for access_token + refresh_token
   ↓
7. App X uses access_token to call API (expires after 15 min)
   ↓
8. When access_token expires, use refresh_token to get new access_token
```

### 3. Comparison Table

| Aspect | API Keys | OAuth2 |
|--------|----------|--------|
| **Target audience** | Internal systems, server-to-server | Third-party apps, user-facing |
| **Authorization flow** | Admin generates key, copy-paste | User authorizes via consent screen |
| **Access scope** | Tenant-wide (all tenant data) | User-specific (authorizing user's data) |
| **Token lifetime** | Long-lived (never expires unless set) | Short-lived (15 min access, 30 day refresh) |
| **Setup complexity** | Simple (1 step: generate key) | Complex (OAuth2 flow, token management) |
| **Security model** | Shared secret (must be stored securely) | No shared secrets (user never sees tokens) |
| **Revocation** | Admin revokes in UI | User revokes in OAuth2 consent screen |
| **Best for** | Automated workflows, batch jobs | Mobile apps, browser extensions |
| **Token format** | `tc_live_abc123...` (opaque string) | JWT (access_token), opaque (refresh_token) |

### 4. Hybrid Use Cases

**Scenario:** Mixed requirements (internal + external)

**Example:** SaaS platform with both:
- Internal ETL pipeline (uses API key)
- Customer-facing mobile app (uses OAuth2)

**Decision:** Support both simultaneously
- ETL pipeline uses tenant API key (tenant-wide access)
- Mobile app users use OAuth2 (user-specific access)
- API gateway validates both (checks Authorization header format)

**Implementation:**
```python
def authenticate_request(request):
    auth_header = request.headers.get('Authorization')

    if auth_header.startswith('Bearer sk_'):
        # API key authentication
        return api_key_validator.validate_key(auth_header.split(' ')[1])
    elif auth_header.startswith('Bearer eyJ'):
        # OAuth2 JWT token authentication
        return oauth2_provider.validate_token(auth_header.split(' ')[1])
    else:
        return None  # Invalid auth
```

---

## Consequences

### Positive

✅ **Clear guidance:** Customers know which method to use based on use case

✅ **Flexibility:** Supports both internal (API keys) and external (OAuth2) integrations

✅ **Security:** OAuth2 for third-party apps limits damage if token stolen (short-lived)

✅ **Simplicity:** API keys for internal systems avoid OAuth2 complexity

✅ **User control:** OAuth2 allows end-users to authorize/revoke access (not just admin)

✅ **Granular scopes:** OAuth2 supports user-specific permissions (can't with API keys)

### Negative

❌ **Confusion:** Customers may not understand when to use which method

❌ **Documentation burden:** Must document both methods clearly (separate guides)

❌ **Complexity:** Supporting both methods increases API gateway complexity

❌ **Security education:** Customers must understand API key storage best practices

### Mitigations

**Confusion:**
- Clear decision tree in docs (internal system → API key, third-party app → OAuth2)
- Quick-start guides for both methods (separate tutorials)
- Warning in UI: "Use API keys for internal systems only. For third-party apps, use OAuth2."

**Documentation:**
- Separate guides: "API Key Authentication" and "OAuth2 Authentication"
- Code examples for both methods (curl, Python, JavaScript)
- FAQ: "When should I use API keys vs OAuth2?"

**Complexity:**
- API gateway checks token format (sk_* → API key, eyJ* → OAuth2 JWT)
- Shared authentication logic (both produce same AuthContext)

**Security education:**
- Docs emphasize: "Never commit API keys to code"
- Example: Store in environment variable, not hardcoded string
- Link to secret management best practices (AWS Secrets Manager, HashiCorp Vault)

---

## Alternatives Considered

### Alternative 1: API Keys Only (No OAuth2)

**Pros:**
- Simpler implementation (no OAuth2 flow)
- Less documentation needed

**Cons:**
- ❌ **No third-party app support** (can't build mobile apps, browser extensions)
- ❌ **No user-specific scopes** (all access is tenant-wide)
- ❌ **Security risk for third-party apps** (long-lived tokens in user-facing apps)

**Decision:** ❌ Rejected - OAuth2 is essential for third-party integrations

### Alternative 2: OAuth2 Only (No API Keys)

**Pros:**
- Simpler guidance (one authentication method)
- More secure (short-lived tokens)

**Cons:**
- ❌ **Overkill for internal systems** (OAuth2 flow unnecessary for cron jobs)
- ❌ **Complexity burden** (customers must implement OAuth2 for simple ETL)
- ❌ **Developer friction** (OAuth2 is harder to get started than API keys)

**Decision:** ❌ Rejected - API keys are simpler for internal use cases

### Alternative 3: JWT Tokens (Self-Issued)

**Example:** Customer generates JWT with private key, signs requests

**Pros:**
- No API key storage (customer generates token on-demand)
- More secure (short-lived, cryptographically signed)

**Cons:**
- ❌ **Complex setup** (customer must manage private/public key pairs)
- ❌ **Uncommon in REST APIs** (most APIs use API keys or OAuth2)
- ❌ **Developer friction** (requires crypto library, key management)

**Decision:** ❌ Rejected - API keys + OAuth2 are more standard and simpler

### Alternative 4: Basic Auth (Username + Password)

**Example:** `Authorization: Basic base64(username:password)`

**Pros:**
- Simplest implementation (built into HTTP)

**Cons:**
- ❌ **Insecure** (password transmitted with every request, even if base64-encoded)
- ❌ **No scopes** (all-or-nothing access)
- ❌ **No expiration** (password valid until manually changed)
- ❌ **Outdated** (modern APIs use tokens, not passwords)

**Decision:** ❌ Rejected - Insecure and outdated pattern

---

## Related Decisions

- **ADR-0039 (API Versioning):** Both API keys and OAuth2 work with versioned endpoints
- **ADR-0040 (Webhook Retry):** Webhooks use API keys for signature generation (not OAuth2)
- **ADR-0037 (RBAC Implementation):** API key scopes map to RBAC roles (read, write, admin)

---

## Documentation Templates

### Quick Start: API Keys (Internal Systems)

```markdown
# Quick Start: API Key Authentication

Use API keys for **internal systems** (ERP, cron jobs, ETL pipelines).

## Step 1: Generate API Key

1. Log in to dashboard
2. Navigate to **API Settings → API Keys**
3. Click **+ New Key**
4. Select scopes: `read`, `write` (or `admin` for full access)
5. Click **Generate Key**
6. **IMPORTANT:** Copy key immediately (you won't see it again)

## Step 2: Store Key Securely

```bash
# Store in environment variable (NEVER commit to code)
export TRUTH_API_KEY="tc_live_3K7mP9xQ2jR8vN5wL1tY4uA6bC0dE"
```

## Step 3: Make API Request

```bash
curl https://api.truthconstruction.com/v1/observations \
  -H "Authorization: Bearer $TRUTH_API_KEY"
```

**Security Best Practices:**
- Store key in environment variable or secret manager
- Never commit key to code (use .gitignore)
- Rotate keys every 12 months
- Revoke immediately if leaked
```

### Quick Start: OAuth2 (Third-Party Apps)

```markdown
# Quick Start: OAuth2 Authentication

Use OAuth2 for **third-party apps** (mobile apps, browser extensions, SaaS integrations).

## Step 1: Register OAuth2 Client

1. Log in to dashboard
2. Navigate to **API Settings → OAuth2 Clients**
3. Click **+ New Client**
4. Enter redirect URI (e.g., `https://yourapp.com/callback`)
5. Copy `client_id` and `client_secret`

## Step 2: Implement Authorization Flow

```javascript
// 1. Redirect user to authorization page
const authUrl = 'https://api.truthconstruction.com/oauth/authorize' +
  `?client_id=${clientId}` +
  `&redirect_uri=${redirectUri}` +
  `&scopes=read,write`;
window.location.href = authUrl;

// 2. User authorizes, redirected to callback with code
// https://yourapp.com/callback?code=auth_abc123

// 3. Exchange code for tokens
const response = await fetch('https://api.truthconstruction.com/oauth/token', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    client_id: clientId,
    client_secret: clientSecret,
    code: authorizationCode,
    redirect_uri: redirectUri
  })
});

const {access_token, refresh_token} = await response.json();

// 4. Use access_token for API requests
const observations = await fetch('https://api.truthconstruction.com/v1/observations', {
  headers: {'Authorization': `Bearer ${access_token}`}
});
```

## Step 3: Refresh Token When Expired

```javascript
// Access token expires after 15 minutes, use refresh token
const response = await fetch('https://api.truthconstruction.com/oauth/refresh', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    client_id: clientId,
    client_secret: clientSecret,
    refresh_token: refreshToken
  })
});

const {access_token: newAccessToken} = await response.json();
```
```

---

## References

- [OAuth 2.0 Specification](https://tools.ietf.org/html/rfc6749)
- [RFC 6750 (Bearer Token Usage)](https://tools.ietf.org/html/rfc6750)
- [Stripe API Authentication](https://stripe.com/docs/api/authentication)
- [GitHub API Authentication](https://docs.github.com/en/rest/overview/authenticating-to-the-rest-api)
