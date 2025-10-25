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
- **Research:** Routes citation queries to get_entities_handler
- **E-Commerce:** Routes invoice reconciliation to trigger_reconciliation_handler
- **HR SaaS:** Routes W-2 upload to create_upload_handler
- **Insurance:** Routes claim queries to get_observations_handler

**Pattern:** APIRouter is domain-agnostic, maps any HTTP request to appropriate handler regardless of domain.

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
