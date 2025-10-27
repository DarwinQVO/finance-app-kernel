# APIRouter (OL Primitive)

**Domain:** Cross-Cutting (API Management)
**Layer:** Objective Layer (OL)
**Vertical:** 5.5 Public API Contracts
**Created:** 2025-10-25

---

## Purpose

Maps HTTP requests to business logic handlers, validates request payloads against OpenAPI schema, serializes responses to JSON, and handles errors in RFC 7807 Problem Details format.

---

## Responsibilities

1. **Route Mapping:** Map HTTP method + path to handler function (e.g., GET /v1/observations → get_observations_handler)
2. **Request Validation:** Validate request body/query params against OpenAPI schema
3. **Response Serialization:** Convert business logic output to JSON (with proper HTTP status codes)
4. **Error Handling:** Catch exceptions, normalize to RFC 7807 Problem Details
5. **Content Negotiation:** Support JSON (required) and optionally CSV/XML for list endpoints

---

## Interface

```python
class APIRouter:
    def __init__(self, openapi_spec: dict):
        self.openapi_spec = openapi_spec
        self.routes = self._build_routes()

    def route(self, request: HTTPRequest, auth_context: AuthContext) -> HTTPResponse:
        """
        Route request to appropriate handler.

        Steps:
        1. Match request method + path to route definition
        2. Validate request payload against OpenAPI schema
        3. Call handler function with validated input
        4. Serialize response to JSON
        5. Return HTTPResponse with status code, headers, body

        Returns HTTP 404 if route not found.
        Returns HTTP 400 if request validation fails.
        Returns HTTP 500 for internal errors.
        """
        pass

    def _build_routes(self) -> dict:
        """
        Build route map from OpenAPI spec.

        Example:
        {
          "GET /v1/observations": get_observations_handler,
          "POST /v1/uploads": create_upload_handler,
          "PATCH /v1/entities/:id": update_entity_handler
        }
        """
        pass

    def validate_request(self, request: HTTPRequest, schema: dict) -> Union[dict, HTTPResponse]:
        """
        Validate request body/query params against JSON schema.

        Returns validated data (dict) if valid.
        Returns HTTP 400 response if validation fails.
        """
        pass

    def serialize_response(self, data: Any, status_code: int = 200) -> HTTPResponse:
        """
        Serialize response data to JSON.

        Sets:
        - Content-Type: application/json
        - Status code (default 200)
        - Body: JSON-serialized data
        """
        pass

    def handle_error(self, error: Exception, request: HTTPRequest) -> HTTPResponse:
        """
        Handle exceptions and return RFC 7807 Problem Details.

        Error Types:
        - ValidationError → HTTP 400
        - NotFoundError → HTTP 404
        - PermissionError → HTTP 403
        - RateLimitError → HTTP 429
        - InternalError → HTTP 500
        """
        pass
```

---

## Data Model

**Route (In-Memory):**

```python
@dataclass
class Route:
    method: str  # GET, POST, PATCH, DELETE
    path: str  # e.g., "/v1/observations"
    handler: Callable  # Handler function
    request_schema: dict  # JSON schema for request body
    response_schema: dict  # JSON schema for response body
    auth_required: bool = True
    scopes: list[str] = field(default_factory=list)  # Required scopes (e.g., ['read'])
```

**HTTPRequest (In-Memory):**

```python
@dataclass
class HTTPRequest:
    method: str
    path: str
    headers: dict
    query_params: dict
    body: Optional[dict]
    auth_context: Optional[AuthContext]
```

**HTTPResponse (In-Memory):**

```python
@dataclass
class HTTPResponse:
    status: int
    headers: dict
    body: dict
```

---

## Implementation Notes

### OpenAPI Spec Integration

APIRouter is driven by OpenAPI 3.0 specification:

```yaml
# openapi.yaml (excerpt)
openapi: 3.0.0
info:
  title: Truth Construction API
  version: 1.0.0

paths:
  /v1/observations:
    get:
      summary: List observations
      operationId: list_observations
      parameters:
        - name: upload_id
          in: query
          required: false
          schema:
            type: string
      responses:
        '200':
          description: List of observations
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Observation'

  /v1/uploads:
    post:
      summary: Create upload
      operationId: create_upload
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UploadRequest'
      responses:
        '201':
          description: Upload created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Upload'

components:
  schemas:
    Observation:
      type: object
      required: [observation_id, upload_id, type]
      properties:
        observation_id:
          type: string
        upload_id:
          type: string
        type:
          type: string
        data:
          type: object

    UploadRequest:
      type: object
      required: [file_path, parser_type]
      properties:
        file_path:
          type: string
        parser_type:
          type: string
          enum: [bank_statement, invoice, lab_report, legal_contract]
```

### Handler Function Pattern

```python
# Handler function for GET /v1/observations
def get_observations_handler(query_params: dict, auth_context: AuthContext) -> list[dict]:
    """
    Business logic for listing observations.

    Returns list of observations (will be serialized to JSON by APIRouter).
    Raises NotFoundError if upload_id not found.
    Raises PermissionError if auth_context lacks 'read' scope.
    """
    upload_id = query_params.get('upload_id')
    tenant_id = auth_context.tenant_id

    # Check permissions
    if 'read' not in auth_context.scopes:
        raise PermissionError("Missing 'read' scope")

    # Query database
    observations = db.query("""
        SELECT observation_id, upload_id, type, data
        FROM observations
        WHERE tenant_id = %s AND ($1 IS NULL OR upload_id = $1)
    """, (tenant_id, upload_id))

    return observations

# Handler function for POST /v1/uploads
def create_upload_handler(body: dict, auth_context: AuthContext) -> dict:
    """
    Business logic for creating upload.

    Returns upload record (will be serialized to JSON by APIRouter).
    Raises ValidationError if file_path invalid.
    Raises PermissionError if auth_context lacks 'write' scope.
    """
    if 'write' not in auth_context.scopes:
        raise PermissionError("Missing 'write' scope")

    file_path = body['file_path']
    parser_type = body['parser_type']
    tenant_id = auth_context.tenant_id

    # Create upload record
    upload_id = str(uuid4())
    db.execute("""
        INSERT INTO uploads (upload_id, tenant_id, file_path, parser_type, status)
        VALUES (%s, %s, %s, %s, 'queued')
    """, (upload_id, tenant_id, file_path, parser_type))

    # Queue for parsing
    queue.publish('parse_queue', {'upload_id': upload_id})

    return {
        'upload_id': upload_id,
        'status': 'queued',
        'created_at': datetime.utcnow().isoformat()
    }
```

---

## Example Usage

```python
# Example 1: Route GET /v1/observations
openapi_spec = load_openapi_spec('openapi.yaml')
router = APIRouter(openapi_spec)

request = HTTPRequest(
    method="GET",
    path="/v1/observations",
    query_params={"upload_id": "upl_abc123"},
    auth_context=AuthContext(tenant_id="tenant_acme", scopes=["read"])
)

response = router.route(request, request.auth_context)
# Response:
# Status: 200
# Headers: Content-Type: application/json
# Body: [{"observation_id": "obs_1", "upload_id": "upl_abc123", ...}]

# Example 2: Route POST /v1/uploads
request = HTTPRequest(
    method="POST",
    path="/v1/uploads",
    body={"file_path": "s3://bucket/statement.pdf", "parser_type": "bank_statement"},
    auth_context=AuthContext(tenant_id="tenant_acme", scopes=["write"])
)

response = router.route(request, request.auth_context)
# Response:
# Status: 201
# Headers: Content-Type: application/json
# Body: {"upload_id": "upl_xyz789", "status": "queued", "created_at": "2025-10-25T10:15:23Z"}

# Example 3: Validation error
request = HTTPRequest(
    method="POST",
    path="/v1/uploads",
    body={"file_path": "s3://bucket/statement.pdf"},  # Missing parser_type
    auth_context=AuthContext(tenant_id="tenant_acme", scopes=["write"])
)

response = router.route(request, request.auth_context)
# Response:
# Status: 400
# Body: {
#   "type": "https://docs.example.com/errors/validation-error",
#   "title": "Validation Error",
#   "status": 400,
#   "detail": "Missing required field: parser_type",
#   "instance": "/v1/uploads"
# }

# Example 4: Permission error
request = HTTPRequest(
    method="POST",
    path="/v1/uploads",
    body={"file_path": "s3://bucket/statement.pdf", "parser_type": "bank_statement"},
    auth_context=AuthContext(tenant_id="tenant_acme", scopes=["read"])  # Missing 'write'
)

response = router.route(request, request.auth_context)
# Response:
# Status: 403
# Body: {
#   "type": "https://docs.example.com/errors/permission-denied",
#   "title": "Permission Denied",
#   "status": 403,
#   "detail": "Missing 'write' scope"
# }
```

---

## Domain Applicability

**Universal Across All Domains:**

- **Finance:** Routes bank statement upload requests to create_upload_handler
- **Healthcare:** Routes lab report queries to get_observations_handler
- **Legal:** Routes contract entity retrieval to get_entities_handler
- **Research (RSRCH - Utilitario):** Routes founder fact queries (GET /v1/facts?subject_entity=Sam+Altman) to get_facts_handler, routes entity resolution (POST /v1/entities/resolve) to resolve_entity_handler
- **E-Commerce:** Routes invoice reconciliation to trigger_reconciliation_handler
- **HR SaaS:** Routes W-2 upload to create_upload_handler
- **Insurance:** Routes claim queries to get_observations_handler

**Pattern:** APIRouter is domain-agnostic, maps any HTTP request to appropriate handler regardless of domain.

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Route bank statement upload and transaction query requests to business logic handlers
**Example:** Client calls POST /v1/uploads with {file_path: "s3://bucket/statement.pdf", parser_type: "bank_statement"} → APIRouter.route() validates request against OpenAPI schema (parser_type required, enum: [bank_statement, invoice, ...]) → Calls create_upload_handler(body, auth_context) → Handler creates upload record with status='queued' → APIRouter.serialize_response() returns HTTP 201 with {upload_id: "upl_abc123", status: "queued"} → Later client calls GET /v1/observations?upload_id=upl_abc123 → APIRouter routes to get_observations_handler → Returns list of transaction observations (JSON serialized)
**Operations:** route (match method+path to handler), validate_request (OpenAPI schema validation), serialize_response (JSON encoding), handle_error (RFC 7807 Problem Details)
**Routes handled:** POST /v1/uploads (create upload), GET /v1/observations (list transactions), PATCH /v1/entities/:id (update merchant), GET /v1/canonical (query canonical records)
**Performance:** <20ms p95 total router overhead (route matching O(1), schema validation <10ms, JSON serialization <5ms)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Route patient lab result queries and prescription upload requests to EHR business logic
**Example:** EHR system calls GET /v1/observations?type=lab_result&patient_id=pat_123 → APIRouter.route() matches to get_observations_handler → Handler checks auth_context.scopes contains "read:lab_results" → Queries observations table filtered by type='lab_result', patient_id → APIRouter.serialize_response() returns HTTP 200 with [{observation_id: "obs_1", type: "lab_result", data: {test: "Glucose", value: "95", unit: "mg/dL"}}] → If EHR calls POST /v1/observations with invalid data (missing required field 'test_name') → APIRouter.validate_request() fails against OpenAPI schema → Returns HTTP 400 RFC 7807 error {type: "validation-error", detail: "Missing required field: test_name"}
**Operations:** OpenAPI schema validation (FHIR-compatible observation schema), scope enforcement (read:lab_results, read:prescriptions), error handling (HIPAA-compliant error messages, no PHI in logs)
**Routes handled:** GET /v1/observations (lab results), POST /v1/observations (add prescription), GET /v1/patients/:id (patient summary), PATCH /v1/observations/:id (update lab result)
**Performance:** <20ms p95 routing overhead, critical for high-volume EHR queries (1000+ req/min)
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ Legal
**Use case:** Route contract document upload and entity extraction requests to legal case management
**Example:** Law firm uploads contract via POST /v1/uploads {file_path: "s3://contracts/nda.pdf", parser_type: "legal_contract"} → APIRouter.route() validates against OpenAPI schema (parser_type enum includes 'legal_contract') → Calls create_upload_handler → Handler creates upload, queues for contract parsing → Returns HTTP 201 {upload_id: "upl_xyz789"} → Later firm queries extracted entities via GET /v1/entities?upload_id=upl_xyz789&type=party → APIRouter routes to get_entities_handler → Returns entities (parties, dates, clauses) JSON serialized
**Operations:** Request validation (contract-specific schemas for parties, clauses, obligations), response serialization (nested entity structures), error handling (bar association compliance logging)
**Routes handled:** POST /v1/uploads (upload contracts), GET /v1/entities (extract parties, clauses), PATCH /v1/entities/:id (correct entity name), GET /v1/canonical (canonical party names)
**Performance:** <20ms p95 routing overhead, <10ms schema validation for complex nested contract entities
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ RSRCH (Utilitario Research)
**Use case:** Route founder fact queries and entity resolution requests to VC research database
**Example:** VC analyst queries founder facts via GET /v1/facts?subject_entity=Sam+Altman&fact_type=investment → APIRouter.route() matches to get_facts_handler → Validates query params against OpenAPI schema (subject_entity required string, fact_type enum: [investment, board_seat, company_launch, ...]) → Handler queries facts table filtered by subject_entity='@sama' (resolved alias), fact_type='investment' → APIRouter.serialize_response() returns HTTP 200 with [{fact_id: "fact_1", subject_entity: "@sama", claim: "Invested $375M in OpenAI", source_url: "https://techcrunch.com/...", publication_date: "2024-02-25"}] → If analyst posts invalid fact (POST /v1/facts missing 'claim' field) → APIRouter.validate_request() rejects with HTTP 400 {detail: "Missing required field: claim"}
**Operations:** OpenAPI schema for facts (subject_entity, claim, fact_type, source_url, confidence), entity resolution routing (POST /v1/entities/resolve maps "Sam Altman" → "@sama"), response pagination (limit/offset for 1000+ fact results), error handling (RFC 7807 for API consumers)
**Routes handled:** GET /v1/facts (query facts), POST /v1/facts (add fact), POST /v1/entities/resolve (resolve entity aliases), GET /v1/companies (company profiles), GET /v1/relationships (founder-company relationships)
**Performance:** <20ms p95 routing overhead critical for high-frequency fact queries (100+ req/sec during diligence), schema validation caching reduces overhead
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ E-commerce
**Use case:** Route invoice upload and product catalog queries to merchant order processing
**Example:** Merchant uploads invoice via POST /v1/uploads {file_path: "s3://invoices/inv_2024_03.pdf", parser_type: "invoice"} → APIRouter.route() validates schema (parser_type enum: [invoice, receipt, catalog]) → Calls create_upload_handler → Returns HTTP 201 {upload_id: "upl_inv123", status: "queued"} → Merchant queries products via GET /v1/observations?type=product&sku=IPHONE15 → APIRouter routes to get_observations_handler → Handler validates scope "read:products" → Returns product observations → If merchant sends malformed request (POST /v1/reconciliations missing 'upload_ids' array) → APIRouter.validate_request() catches, returns HTTP 400 RFC 7807 error {detail: "Field 'upload_ids' must be array"}
**Operations:** Schema validation (invoice, product, order schemas), content negotiation (JSON for API, CSV for batch exports via GET /v1/observations?format=csv), error handling (PCI DSS compliant error messages, no payment card data in logs)
**Routes handled:** POST /v1/uploads (invoices), GET /v1/observations (products), POST /v1/reconciliations (match invoices to orders), GET /v1/canonical (canonical product names)
**Performance:** <20ms p95 routing overhead, <10ms schema validation for nested product catalogs (100+ variants per product)
**Status:** ✅ Conceptually validated via examples in this doc

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (HTTP routing with OpenAPI schema validation is universal pattern, no domain-specific code in router logic)
**Reusability:** High (same route/validate/serialize operations work for bank statements, lab results, contracts, founder facts, invoices; only OpenAPI schemas and handler functions differ)

---

## Simplicity Profiles

### Personal Profile (20 LOC)

**Contexto del Usuario:**
Aplicación personal con 3 endpoints fijos: upload PDF, get transactions, get dashboard. Hardcoded if/elif routing (no framework needed).

**Implementation:**
```python
def handle_request(method, path, body):
    if method == "POST" and path == "/upload":
        return create_upload(body)
    elif method == "GET" and path == "/transactions":
        return get_transactions()
    elif method == "GET" and path == "/dashboard":
        return get_dashboard()
    else:
        return {"error": "Not found"}, 404
```

**Características Incluidas:**
- ✅ Route matching básico (if/elif)
- ✅ 3 endpoints hardcoded

**Características NO Incluidas:**
- ❌ OpenAPI schema validation (YAGNI: inputs trusted)
- ❌ Content negotiation (solo JSON)
- ❌ Middleware (no authentication layer needed)

**Configuración:**
```python
# No config needed - routes hardcoded
```

**Performance:**
- Latency: <1ms (if/elif comparison)

**Upgrade Triggers:**
- Si necesita >10 endpoints → Small Business (dict-based routing)
- Si necesita validación → Small Business (schema validation)

---

### Small Business Profile (80 LOC)

**Contexto del Usuario:**
Firma con 15 endpoints. Usa dict para routing + basic middleware (auth, logging).

**Implementation:**
```python
ROUTES = {
    ("GET", "/v1/transactions"): get_transactions_handler,
    ("POST", "/v1/uploads"): create_upload_handler,
    ("GET", "/v1/dashboard"): get_dashboard_handler,
    # ... 12 more endpoints
}

def route_request(method, path, headers, body):
    # Middleware: Auth
    auth_context = validate_api_key(headers.get("Authorization"))
    if not auth_context:
        return {"error": "Unauthorized"}, 401

    # Route lookup
    handler = ROUTES.get((method, path))
    if not handler:
        return {"error": "Not found"}, 404

    # Call handler
    try:
        result = handler(body, auth_context)
        return result, 200
    except ValueError as e:
        return {"error": str(e)}, 400
```

**Características Incluidas:**
- ✅ Dict-based routing (O(1) lookup)
- ✅ Basic middleware (auth, error handling)
- ✅ 15+ endpoints

**Características NO Incluidas:**
- ❌ OpenAPI validation (manual validation in handlers)
- ❌ Versioning (no /v2 paths yet)

**Configuración:**
```yaml
api_router:
  routes_file: "routes.py"
```

**Performance:**
- Latency: <2ms (dict lookup)

**Upgrade Triggers:**
- Si >50 endpoints → Enterprise (OpenAPI spec)
- Si necesita versioning (/v1, /v2) → Enterprise

---

### Enterprise Profile (400 LOC)

**Contexto del Usuario:**
FinTech con 100+ endpoints, API versioning (/v1, /v2), OpenAPI schema validation, rate limiting per endpoint.

**Implementation:**
```python
from fastapi import FastAPI, Request
from pydantic import ValidationError

app = FastAPI()

@app.post("/v1/uploads")
async def create_upload(request: UploadRequest, auth: AuthContext = Depends(validate_token)):
    """
    OpenAPI-driven routing with automatic validation.
    FastAPI validates UploadRequest against Pydantic model.
    """
    result = await upload_manager.create(request, auth)
    return result

@app.get("/v2/transactions")
@rate_limit(max_requests=100, window_seconds=60)
async def get_transactions_v2(
    filters: TransactionFilters = Query(),
    auth: AuthContext = Depends(validate_token)
):
    """
    Versioned endpoint (/v2) with rate limiting.
    """
    result = await transaction_query.search(filters, auth)
    return result
```

**Características Incluidas:**
- ✅ OpenAPI schema validation (FastAPI/Pydantic)
- ✅ API versioning (/v1, /v2 paths)
- ✅ Rate limiting per endpoint
- ✅ Auto-generated OpenAPI docs

**Características NO Incluidas:**
- ❌ GraphQL (REST-only)

**Configuración:**
```yaml
api_router:
  framework: "fastapi"
  openapi_version: "3.1.0"
  rate_limits:
    default: 1000
    endpoints:
      POST /v1/uploads: 100
```

**Performance:**
- Latency: <5ms (FastAPI overhead)
- Throughput: 10K req/sec

**No Further Tiers:**
- Scale horizontally (load balancer)

---

## Related Primitives

- **APIGateway (OL):** Calls APIRouter after authentication/rate limiting
- **UploadManager (OL, from 1.1):** Used by create_upload_handler to create upload records
- **ObservationStore (OL, from 1.2):** Used by get_observations_handler to retrieve observations
- **EntityResolver (OL, from 2.1):** Used by get_entities_handler to retrieve entities
- **ReconciliationEngine (OL, from 3.1):** Used by trigger_reconciliation_handler

---

## Testing

```python
# Unit Test: Route matching
def test_route_matching():
    router = APIRouter(openapi_spec)
    request = HTTPRequest(method="GET", path="/v1/observations")
    route = router._match_route(request)
    assert route.handler == get_observations_handler

# Unit Test: Request validation success
def test_request_validation_success():
    router = APIRouter(openapi_spec)
    request = HTTPRequest(
        method="POST",
        path="/v1/uploads",
        body={"file_path": "s3://bucket/file.pdf", "parser_type": "bank_statement"}
    )
    validated_data = router.validate_request(request, upload_request_schema)
    assert validated_data['parser_type'] == 'bank_statement'

# Unit Test: Request validation failure
def test_request_validation_failure():
    router = APIRouter(openapi_spec)
    request = HTTPRequest(
        method="POST",
        path="/v1/uploads",
        body={"file_path": "s3://bucket/file.pdf"}  # Missing parser_type
    )
    response = router.validate_request(request, upload_request_schema)
    assert response.status == 400
    assert "parser_type" in response.body['detail']

# Integration Test: End-to-end routing
def test_end_to_end_routing():
    router = APIRouter(openapi_spec)
    request = HTTPRequest(
        method="GET",
        path="/v1/observations",
        query_params={"upload_id": "upl_abc123"},
        auth_context=AuthContext(tenant_id="tenant_acme", scopes=["read"])
    )
    response = router.route(request, request.auth_context)
    assert response.status == 200
    assert isinstance(response.body, list)
```

---

## Performance

- **Route Matching:** O(1) (hash map lookup)
- **Request Validation:** p95 < 10ms (JSON schema validation)
- **Response Serialization:** p95 < 5ms (JSON encoding)
- **Total Router Overhead:** p95 < 20ms

**Optimization:** Cache compiled JSON schemas (avoid recompiling on every request).

---

## Metadata

- **Lines of Code:** ~400 (Python implementation)
- **Dependencies:** OpenAPI spec, JSON schema validator (jsonschema), handler functions, database
- **Deployment:** Runs as part of API service (Node.js, Python, Go)
