# APIGateway (OL Primitive)

**Domain:** Cross-Cutting (API Management)
**Layer:** Objective Layer (OL)
**Vertical:** 5.5 Public API Contracts
**Created:** 2025-10-25

---

## Purpose

Entry point for all public API requests. Handles authentication (API keys, OAuth2), rate limiting, CORS, request logging, and routes validated requests to APIRouter.

---

## Responsibilities

1. **Authentication:** Validate API keys (via APIKeyValidator) or OAuth2 tokens (via OAuth2Provider)
2. **Rate Limiting:** Check per-tenant, per-endpoint rate limits (via RateLimiter primitive)
3. **CORS Handling:** Add CORS headers for browser-based API clients
4. **Request Logging:** Log all requests for audit trail (via AuditLogger primitive)
5. **Error Normalization:** Return RFC 7807 Problem Details for errors

---

## Interface

```python
class APIGateway:
    def __init__(self, api_key_validator: APIKeyValidator,
                 oauth2_provider: OAuth2Provider,
                 rate_limiter: RateLimiter,
                 audit_logger: AuditLogger):
        self.api_key_validator = api_key_validator
        self.oauth2_provider = oauth2_provider
        self.rate_limiter = rate_limiter
        self.audit_logger = audit_logger

    def handle_request(self, request: HTTPRequest) -> HTTPResponse:
        """
        Main entry point for API requests.

        Steps:
        1. Authenticate request (API key or OAuth2 token)
        2. Check rate limits
        3. Log request
        4. Forward to APIRouter
        5. Add rate limit headers to response
        6. Return response

        Returns HTTP 401 if authentication fails.
        Returns HTTP 429 if rate limit exceeded.
        Returns HTTP 500 for internal errors (with RFC 7807 Problem Details).
        """
        pass

    def authenticate(self, request: HTTPRequest) -> Optional[AuthContext]:
        """
        Extract and validate authentication credentials.

        Checks:
        - Authorization header (Bearer token)
        - API key format (tc_live_* or tc_test_*)
        - OAuth2 token validity (expiration, scopes)

        Returns AuthContext with tenant_id, user_id, scopes.
        Returns None if authentication fails.
        """
        pass

    def check_rate_limit(self, tenant_id: str, endpoint: str, tier: str) -> RateLimitResult:
        """
        Check rate limit for tenant + endpoint.

        Uses sliding window algorithm (Redis counters).
        Checks both hourly limit and burst limit (per-minute).

        Returns RateLimitResult with:
        - allowed: bool
        - limit: int (max requests in window)
        - remaining: int (requests remaining)
        - reset: int (Unix timestamp when limit resets)
        """
        pass

    def add_cors_headers(self, response: HTTPResponse) -> HTTPResponse:
        """
        Add CORS headers to response for browser-based clients.

        Headers:
        - Access-Control-Allow-Origin: * (or specific origin)
        - Access-Control-Allow-Methods: GET, POST, PATCH, DELETE
        - Access-Control-Allow-Headers: Authorization, Content-Type
        - Access-Control-Max-Age: 86400 (24 hours)
        """
        pass

    def log_request(self, request: HTTPRequest, response: HTTPResponse, auth_context: AuthContext):
        """
        Log API request for audit trail.

        Logs:
        - Timestamp, tenant_id, endpoint, HTTP method, status code
        - Response time, IP address, user agent
        - Rate limit remaining
        """
        pass

    def normalize_error(self, error: Exception) -> HTTPResponse:
        """
        Normalize error to RFC 7807 Problem Details format.

        Example:
        {
          "type": "https://docs.example.com/errors/rate-limit-exceeded",
          "title": "Rate Limit Exceeded",
          "status": 429,
          "detail": "You have exceeded your rate limit of 1000 requests per hour.",
          "instance": "/v1/observations",
          "retry_after": 300
        }
        """
        pass
```

---

## Data Model

**AuthContext (In-Memory):**

```python
@dataclass
class AuthContext:
    tenant_id: str
    user_id: Optional[str]  # None for API keys, set for OAuth2
    scopes: list[str]  # ['read', 'write', 'admin']
    api_key_id: Optional[str]  # Set for API key auth
    token_id: Optional[str]  # Set for OAuth2 auth
```

**RateLimitResult (In-Memory):**

```python
@dataclass
class RateLimitResult:
    allowed: bool
    limit: int  # Max requests in window (e.g., 1000)
    remaining: int  # Requests remaining (e.g., 847)
    reset: int  # Unix timestamp when limit resets
```

---

## Implementation Notes

### Middleware Architecture

APIGateway is typically implemented as middleware in API framework:

```python
# Express.js (Node.js)
app.use(async (req, res, next) => {
    const gateway = new APIGateway(apiKeyValidator, oauth2Provider, rateLimiter, auditLogger);
    const response = await gateway.handle_request(req);

    if (response.status === 401 || response.status === 429) {
        // Authentication or rate limit failed, return early
        return res.status(response.status).json(response.body);
    }

    // Add rate limit headers
    res.set('X-RateLimit-Limit', response.rate_limit.limit);
    res.set('X-RateLimit-Remaining', response.rate_limit.remaining);
    res.set('X-RateLimit-Reset', response.rate_limit.reset);

    // Forward to router
    next();
});
```

### Kong Gateway Integration

Alternatively, use Kong Gateway (API gateway platform):

```yaml
# kong.yml
_format_version: "3.0"

services:
  - name: truth-construction-api
    url: http://api-service:8000
    routes:
      - name: api-v1
        paths:
          - /v1

plugins:
  - name: key-auth  # API key authentication
    config:
      key_names: [Authorization]

  - name: rate-limiting  # Rate limiting
    config:
      hour: 1000
      policy: redis
      redis_host: redis.default.svc.cluster.local

  - name: cors  # CORS handling
    config:
      origins: ["*"]
      methods: ["GET", "POST", "PATCH", "DELETE"]

  - name: request-transformer  # Add tenant_id header
    config:
      add:
        headers:
          - X-Tenant-ID: ${tenant_id}

  - name: file-log  # Request logging
    config:
      path: /var/log/kong/api-requests.log
```

---

## Example Usage

```python
# Example 1: Successful API request
request = HTTPRequest(
    method="GET",
    path="/v1/observations",
    headers={"Authorization": "Bearer tc_live_abc123xyz"}
)

response = api_gateway.handle_request(request)
# Response:
# Status: 200
# Headers: X-RateLimit-Limit: 1000, X-RateLimit-Remaining: 847, X-RateLimit-Reset: 1730000000
# Body: [{"observation_id": "obs_1", ...}]

# Example 2: Authentication failure
request = HTTPRequest(
    method="GET",
    path="/v1/observations",
    headers={"Authorization": "Bearer invalid_key"}
)

response = api_gateway.handle_request(request)
# Response:
# Status: 401
# Body: {
#   "type": "https://docs.example.com/errors/unauthorized",
#   "title": "Unauthorized",
#   "status": 401,
#   "detail": "Invalid API key"
# }

# Example 3: Rate limit exceeded
# (After 1000 requests in 1 hour)
request = HTTPRequest(
    method="GET",
    path="/v1/observations",
    headers={"Authorization": "Bearer tc_live_abc123xyz"}
)

response = api_gateway.handle_request(request)
# Response:
# Status: 429
# Headers: Retry-After: 300
# Body: {
#   "type": "https://docs.example.com/errors/rate-limit-exceeded",
#   "title": "Rate Limit Exceeded",
#   "status": 429,
#   "detail": "You have exceeded your rate limit of 1000 requests per hour.",
#   "retry_after": 300
# }
```

---

## Domain Applicability

**Universal Across All Domains:**

- **Finance:** Bank API clients authenticate, get rate limited per tier
- **Healthcare:** EHR systems use OAuth2 tokens, rate limited per hospital tenant
- **Legal:** Law firm API keys scoped to tenant, CORS headers for web app
- **Research (RSRCH - Utilitario):** VC firm API keys for querying founder facts (GET /v1/facts?subject_entity=Sam+Altman), rate limits per tier (1000 req/hour standard, 10000 req/hour enterprise), webhook authentication for real-time fact notifications
- **E-Commerce:** SaaS platform OAuth2 integration, high rate limits (Enterprise tier)
- **HR SaaS:** API keys for payroll processing, audit logging for compliance
- **Insurance:** Claims processing API, rate limits per insurance company

**Pattern:** APIGateway is domain-agnostic, applies universally to all API clients regardless of use case.

---

## Simplicity Profiles

### Personal Profile (0 LOC)

**Contexto del Usuario:**
Aplicación personal que corre localmente. No necesita API HTTP porque todas las operaciones son llamadas directas a funciones. No hay "clientes externos" que necesiten autenticación o rate limiting.

**Implementation:**
```python
# No implementation needed (0 LOC)
# Personal app = direct function calls, no HTTP API
# Example: process_upload(file_path) instead of POST /v1/uploads
```

**Características Incluidas:**
- ✅ Ninguna (API Gateway no necesario para apps locales)

**Características NO Incluidas:**
- ❌ HTTP server (YAGNI: llamadas directas a funciones)
- ❌ Authentication (YAGNI: usuario único, trusted environment)
- ❌ Rate limiting (YAGNI: no hay múltiples clientes)
- ❌ CORS (YAGNI: no hay navegador web)

**Configuración:**
```python
# No config needed
```

**Performance:**
- Latency: N/A

**Upgrade Triggers:**
- Si necesita API web → Small Business (HTTP server)
- Si necesita cliente móvil → Small Business (REST API)
- Si necesita integración externa → Small Business (API keys)

---

### Small Business Profile (100 LOC)

**Contexto del Usuario:**
Firma con API REST para 10 clientes externos (accounting software, mobile app). Necesitan autenticación básica con API keys y logging simple de requests.

**Implementation:**
```python
# api_gateway.py (Small Business - 100 LOC)
from flask import Flask, request, jsonify

app = Flask(__name__)
API_KEYS = {"client_quickbooks": "key_123", "client_mobile": "key_456"}

def authenticate_request():
    """Middleware: Validate API key from Authorization header."""
    auth_header = request.headers.get("Authorization")
    if not auth_header or not auth_header.startswith("Bearer "):
        return None

    api_key = auth_header.replace("Bearer ", "")
    # Simple lookup in hardcoded dict
    for client_id, key in API_KEYS.items():
        if key == api_key:
            return {"client_id": client_id}
    return None

@app.before_request
def gateway_middleware():
    """Run before every request: auth + logging."""
    # 1. Authenticate
    auth_context = authenticate_request()
    if not auth_context:
        return jsonify({"error": "Unauthorized"}), 401

    # 2. Log request (simple print to stdout)
    print(f"[API] {request.method} {request.path} - Client: {auth_context['client_id']}")

    # 3. Store auth context for handlers
    request.auth_context = auth_context

@app.route("/v1/transactions", methods=["GET"])
def get_transactions():
    """Example endpoint - auth already checked by middleware."""
    # Business logic here
    return jsonify({"transactions": []})

@app.route("/v1/uploads", methods=["POST"])
def create_upload():
    """Example endpoint - auth already checked by middleware."""
    # Business logic here
    return jsonify({"upload_id": "upload_123"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**Características Incluidas:**
- ✅ HTTP server (Flask)
- ✅ API key authentication (hardcoded dict)
- ✅ Request logging (stdout)
- ✅ Middleware pattern (before_request)

**Características NO Incluidas:**
- ❌ Rate limiting (SQLite-based para Small Business es overkill)
- ❌ OAuth2 (solo API keys)
- ❌ Circuit breakers (no hay microservicios)
- ❌ Metrics collection (logs suficientes)

**Configuración:**
```yaml
api_gateway:
  host: "0.0.0.0"
  port: 5000
  api_keys:
    client_quickbooks: "key_123"
    client_mobile: "key_456"
```

**Performance:**
- Latency: <5ms (middleware overhead)
- Throughput: ~1K req/sec (Flask single process)

**Upgrade Triggers:**
- Si >1K req/sec → Enterprise (async server)
- Si necesita rate limiting → Enterprise (Redis-based)
- Si >20 clientes → Enterprise (database-backed API keys)

---

### Enterprise Profile (500 LOC)

**Contexto del Usuario:**
FinTech con 10K clientes API, rate limiting estricto (1K req/min por tenant), circuit breakers para microservicios internos, métricas Prometheus, CORS para web apps.

**Implementation:**
```python
# api_gateway.py (Enterprise - 500 LOC)
from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.cors import CORSMiddleware
import redis
import time

app = FastAPI()

# CORS for web clients
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)

redis_client = redis.Redis(host="localhost", port=6379)

@app.middleware("http")
async def gateway_middleware(request: Request, call_next):
    """
    Full gateway pipeline:
    1. Authenticate (API key or OAuth2)
    2. Rate limit check
    3. Log request
    4. Forward to router
    5. Add rate limit headers
    """
    start_time = time.time()

    # 1. Authenticate
    auth_context = await authenticate(request)
    if not auth_context:
        return JSONResponse(
            status_code=401,
            content={"type": "about:blank", "title": "Unauthorized", "status": 401}
        )

    # 2. Rate limiting (Redis-based)
    tenant_id = auth_context["tenant_id"]
    rate_limit_key = f"rate_limit:{tenant_id}:{int(time.time() / 60)}"  # Per minute

    current_count = redis_client.incr(rate_limit_key)
    if current_count == 1:
        redis_client.expire(rate_limit_key, 60)  # Expire after 1 minute

    limit = 1000  # 1K req/min
    if current_count > limit:
        return JSONResponse(
            status_code=429,
            content={
                "type": "about:blank",
                "title": "Rate Limit Exceeded",
                "status": 429,
                "detail": f"Rate limit of {limit} requests per minute exceeded"
            },
            headers={
                "X-RateLimit-Limit": str(limit),
                "X-RateLimit-Remaining": "0",
                "Retry-After": "60"
            }
        )

    # 3. Store auth context for downstream handlers
    request.state.auth = auth_context

    # 4. Forward to router (call_next executes the actual endpoint)
    response = await call_next(request)

    # 5. Add rate limit headers
    response.headers["X-RateLimit-Limit"] = str(limit)
    response.headers["X-RateLimit-Remaining"] = str(limit - current_count)

    # 6. Log request with latency
    latency_ms = (time.time() - start_time) * 1000
    print(f"[API] {request.method} {request.url.path} - Tenant: {tenant_id} - {latency_ms:.2f}ms")

    # 7. Emit Prometheus metrics
    emit_metric("api_request_duration_ms", latency_ms, {"endpoint": request.url.path})

    return response

async def authenticate(request: Request):
    """
    Authenticate via API key or OAuth2 token.
    Supports both authentication methods.
    """
    auth_header = request.headers.get("Authorization")
    if not auth_header:
        return None

    if auth_header.startswith("Bearer tc_"):
        # API Key authentication
        api_key = auth_header.replace("Bearer ", "")
        return await api_key_validator.validate_key(api_key)
    elif auth_header.startswith("Bearer ey"):
        # OAuth2 JWT token
        token = auth_header.replace("Bearer ", "")
        return await oauth2_provider.validate_token(token)

    return None
```

**Características Incluidas:**
- ✅ FastAPI async server (high throughput)
- ✅ Dual authentication (API keys + OAuth2)
- ✅ Redis-based rate limiting (1K req/min per tenant)
- ✅ CORS middleware (web client support)
- ✅ RFC 7807 error responses
- ✅ Rate limit headers (X-RateLimit-*)
- ✅ Prometheus metrics

**Características NO Incluidas:**
- ❌ API versioning (manejado por APIRouter)
- ❌ Request transformation (responsibility de router)

**Configuración:**
```yaml
api_gateway:
  server:
    host: "0.0.0.0"
    port: 8000
    workers: 4
  rate_limiting:
    enabled: true
    redis_url: "redis://localhost:6379"
    default_limit: 1000  # per minute
    burst: 1200
  cors:
    allowed_origins:
      - "https://app.example.com"
      - "https://dashboard.example.com"
  authentication:
    api_keys_enabled: true
    oauth2_enabled: true
```

**Performance:**
- Latency: <5ms p95 (middleware overhead)
- Throughput: 50K req/sec (4 workers)
- Rate limiting overhead: <1ms (Redis lookup)

**No Further Tiers:**
- Scale horizontally (load balancer + multiple instances)

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** API gateway for bank API clients authenticating to query transactions
**Example:** Client request GET /v1/transactions with API key "tc_live_abc123" → APIGateway: authenticate() validates key + check_rate_limit() (847/1000 remaining) + log_request() → Forward to APIRouter → Return 200 with rate limit headers
**Operations:** authenticate (API key/OAuth2), check_rate_limit (per-tenant throttling), add_cors_headers, log_request, normalize_error (RFC 7807)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** API gateway for EHR systems using OAuth2 tokens
**Example:** Hospital EHR system request GET /v1/patients with OAuth2 token → APIGateway: authenticate() validates token + scopes ["read:patients"] + check_rate_limit() per hospital tenant → Forward to APIRouter
**Operations:** OAuth2 token validation, scope checking, rate limiting per hospital, HIPAA-compliant audit logging
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ Legal
**Use case:** API gateway for law firm API keys with tenant isolation
**Example:** Law firm API key request GET /v1/cases → APIGateway: authenticate() validates key + TenantIsolator ensures can't access other firms' cases + check_rate_limit() → Forward to APIRouter
**Operations:** API key authentication, tenant isolation enforcement, CORS for web apps, audit logging for compliance
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ RSRCH (Utilitario Research)
**Use case:** API gateway for VC firms querying founder facts
**Example:** VC firm request GET /v1/facts?subject_entity=Sam+Altman with API key → APIGateway: authenticate() validates key + check_rate_limit() (tier: standard 1000 req/hour, enterprise 10000 req/hour) → Forward to APIRouter → Returns founder facts
**Operations:** API key authentication, tiered rate limiting (standard/enterprise), webhook authentication for real-time notifications, CORS for web dashboard
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ E-commerce
**Use case:** API gateway for SaaS platform OAuth2 integration
**Example:** SaaS platform request GET /v1/products with OAuth2 token → APIGateway: authenticate() validates token + check_rate_limit() (high limit for enterprise tier) → Forward to APIRouter
**Operations:** OAuth2 integration, high rate limits for enterprise, API key for integrations, audit logging for billing
**Status:** ✅ Conceptually validated via examples in this doc

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (universal API gateway pattern, no domain-specific logic)
**Reusability:** High (same authenticate/rate_limit/log operations work for banking APIs, healthcare APIs, legal APIs, research APIs, e-commerce APIs)

---

## Related Primitives

- **APIKeyValidator (OL):** Validates API keys, used by APIGateway for authentication
- **OAuth2Provider (OL):** Validates OAuth2 tokens, used by APIGateway for authentication
- **RateLimiter (OL):** Checks rate limits, used by APIGateway for throttling
- **AuditLogger (OL):** Logs API requests, used by APIGateway for compliance
- **APIRouter (OL):** Routes validated requests to business logic, called after APIGateway
- **TenantIsolator (OL, from 5.4):** Ensures API key can't access other tenants' data

---

## Testing

```python
# Unit Test: Authentication success
def test_authenticate_valid_api_key():
    gateway = APIGateway(api_key_validator, oauth2_provider, rate_limiter, audit_logger)
    request = HTTPRequest(headers={"Authorization": "Bearer tc_test_abc123"})
    auth_context = gateway.authenticate(request)
    assert auth_context.tenant_id == "tenant_acme"
    assert "read" in auth_context.scopes

# Unit Test: Authentication failure
def test_authenticate_invalid_api_key():
    gateway = APIGateway(api_key_validator, oauth2_provider, rate_limiter, audit_logger)
    request = HTTPRequest(headers={"Authorization": "Bearer invalid_key"})
    auth_context = gateway.authenticate(request)
    assert auth_context is None

# Unit Test: Rate limit check
def test_rate_limit_exceeded():
    gateway = APIGateway(api_key_validator, oauth2_provider, rate_limiter, audit_logger)
    # Simulate 1001 requests
    for i in range(1001):
        result = gateway.check_rate_limit("tenant_acme", "GET /v1/observations", "starter")
        if i < 1000:
            assert result.allowed == True
        else:
            assert result.allowed == False  # 1001st request rejected

# Integration Test: End-to-end API request
def test_api_request_end_to_end():
    gateway = APIGateway(api_key_validator, oauth2_provider, rate_limiter, audit_logger)
    request = HTTPRequest(
        method="GET",
        path="/v1/observations",
        headers={"Authorization": "Bearer tc_test_abc123"}
    )
    response = gateway.handle_request(request)
    assert response.status == 200
    assert "X-RateLimit-Limit" in response.headers
```

---

## Performance

- **Authentication:** p95 < 20ms (bcrypt validation + Redis cache lookup)
- **Rate Limit Check:** p95 < 5ms (Redis INCR operation)
- **Total Gateway Overhead:** p95 < 30ms (authentication + rate limit + logging)

**Optimization:** Cache validated API keys in Redis (TTL 5 minutes) to reduce database load.

---

## Metadata

- **Lines of Code:** ~300 (Python implementation)
- **Dependencies:** APIKeyValidator, OAuth2Provider, RateLimiter, AuditLogger, Redis, HTTP framework
- **Deployment:** Runs as middleware in API service OR as Kong Gateway plugin
