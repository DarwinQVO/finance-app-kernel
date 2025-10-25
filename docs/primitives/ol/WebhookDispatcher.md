# WebhookDispatcher (OL Primitive)

**Domain:** Cross-Cutting (API Management)
**Layer:** Objective Layer (OL)
**Vertical:** 5.5 Public API Contracts
**Created:** 2025-10-25

---

## Purpose

Listens for system events (upload.completed, observation.created, entity.resolved, reconciliation.completed), queues webhook deliveries to registered consumer endpoints, retries failed deliveries with exponential backoff, and marks failed webhooks after max retries.

---

## Responsibilities

1. **Event Listening:** Subscribe to event stream (Kafka/Redis) for system events
2. **Webhook Matching:** Find all webhooks registered for event type (with optional filters)
3. **Delivery Queueing:** Queue webhook delivery for async processing by worker
4. **Signature Generation:** Generate HMAC-SHA256 signature for payload verification
5. **Retry Logic:** Retry failed deliveries with exponential backoff (5s, 25s, 125s, 625s, 3125s)
6. **Failure Handling:** Mark webhook as "failed" after 5 retries, email admin

---

## Interface

```python
class WebhookDispatcher:
    def __init__(self, event_stream: EventStream, db: Database, queue: Queue, http_client: HTTPClient):
        self.event_stream = event_stream
        self.db = db
        self.queue = queue
        self.http_client = http_client

    def start_listening(self):
        """
        Start listening for events from event stream.

        Subscribes to topics:
        - upload_events (upload.completed, upload.failed)
        - observation_events (observation.created)
        - entity_events (entity.resolved, entity.merged)
        - reconciliation_events (reconciliation.completed)

        Runs indefinitely in background worker.
        """
        pass

    def dispatch_event(self, event_type: str, payload: dict):
        """
        Dispatch event to all registered webhooks.

        Steps:
        1. Query webhooks for event_type (status=active, event in events array)
        2. Apply filters (if webhook has filters, check if payload matches)
        3. Queue delivery for each matching webhook

        Example:
        event_type = "upload.completed"
        payload = {"upload_id": "upl_abc123", "status": "parsed"}
        → Finds 3 webhooks registered for "upload.completed"
        → Queues 3 delivery jobs
        """
        pass

    def queue_delivery(self, webhook: Webhook, event_type: str, payload: dict):
        """
        Queue webhook delivery for async processing.

        Steps:
        1. Generate HMAC-SHA256 signature
        2. Insert delivery record (status=pending)
        3. Publish to queue (Kafka/Redis)

        Delivery record includes:
        - delivery_id, webhook_id, event_type, payload, signature
        - status=pending, retry_count=0, next_retry_at=NULL
        """
        pass

    def deliver_webhook(self, delivery_id: str):
        """
        Worker function to deliver webhook (called by background worker).

        Steps:
        1. Load delivery record + webhook URL/secret
        2. HTTP POST to webhook URL with payload + signature header
        3. Record response (status_code, response_time_ms)
        4. If success (2xx): Mark delivery as success
        5. If retriable error (5xx, timeout): Schedule retry
        6. If non-retriable error (4xx): Mark delivery as failed

        Timeout: 10 seconds (prevent slowloris attacks).
        """
        pass

    def schedule_retry(self, delivery_id: str, retry_count: int, status_code: int, response_time_ms: int):
        """
        Schedule retry with exponential backoff.

        Backoff schedule:
        - Retry 1: 5 seconds
        - Retry 2: 25 seconds (5^2)
        - Retry 3: 125 seconds (5^3)
        - Retry 4: 625 seconds (5^4)
        - Retry 5: 3125 seconds (5^5, ~52 minutes)

        After 5 retries:
        - Mark delivery as failed
        - Mark webhook as failed (status=failed, failure_count++)
        - Email admin with failure details
        """
        pass

    def generate_signature(self, payload: dict, secret: str) -> str:
        """
        Generate HMAC-SHA256 signature for payload verification.

        Steps:
        1. Serialize payload to JSON (canonical format)
        2. Compute HMAC-SHA256(secret, payload_json)
        3. Return "sha256=<hex_digest>"

        Consumer must verify signature before processing webhook.
        """
        pass

    def verify_signature(self, payload: dict, signature: str, secret: str) -> bool:
        """
        Verify webhook signature (used by consumer).

        Steps:
        1. Generate expected signature (same as generate_signature)
        2. Compare with received signature (constant-time comparison)
        3. Return True if match, False otherwise
        """
        pass

    def matches_filters(self, payload: dict, filters: dict) -> bool:
        """
        Check if payload matches webhook filters.

        Example filters:
        {"parser_type": "bank_statement"} → Only trigger for bank statements
        {"status": "parsed"} → Only trigger for parsed uploads (not failed)

        Returns True if all filter keys match payload values.
        """
        pass
```

---

## Data Model

**Webhook (Database Table):**

```sql
CREATE TABLE webhooks (
  webhook_id UUID PRIMARY KEY,
  tenant_id VARCHAR(255) NOT NULL,
  url TEXT NOT NULL,
  secret VARCHAR(255) NOT NULL,  -- HMAC secret
  events TEXT[] NOT NULL,  -- ['upload.completed', 'observation.created']
  filters JSONB,  -- {"parser_type": "bank_statement"}
  status VARCHAR(50) DEFAULT 'active',  -- active, failed, disabled
  last_delivery_at TIMESTAMP NULL,
  failure_count INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP
);
```

**WebhookDelivery (Database Table):**

```sql
CREATE TABLE webhook_deliveries (
  delivery_id UUID PRIMARY KEY,
  webhook_id UUID NOT NULL REFERENCES webhooks(webhook_id),
  event_type VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  signature VARCHAR(255) NOT NULL,  -- sha256=<hex>
  status VARCHAR(50) NOT NULL,  -- pending, success, failed
  status_code INTEGER,  -- HTTP status from webhook endpoint
  response_time_ms INTEGER,
  retry_count INTEGER DEFAULT 0,
  next_retry_at TIMESTAMP,  -- NULL if no retry needed
  delivered_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_webhook_deliveries_webhook ON webhook_deliveries(webhook_id);
CREATE INDEX idx_webhook_deliveries_next_retry ON webhook_deliveries(next_retry_at) WHERE status = 'pending';
```

---

## Implementation Notes

### Event Stream Architecture

```
┌───────────────────────────────────────────────────────────┐
│                      System Events                        │
│  (upload.completed, observation.created, etc.)            │
└───────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│                   Kafka/Redis Stream                      │
│  Topics: upload_events, observation_events, etc.          │
└───────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│              WebhookDispatcher (Listener)                 │
│  - Subscribes to event topics                             │
│  - Dispatches to matching webhooks                        │
│  - Queues delivery jobs                                   │
└───────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│              Webhook Delivery Queue                       │
│  (Kafka/Redis queue, processed by workers)                │
└───────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ Delivery Worker  │ │ Delivery Worker  │ │ Delivery Worker  │
│ (HTTP POST)      │ │ (HTTP POST)      │ │ (HTTP POST)      │
└──────────────────┘ └──────────────────┘ └──────────────────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌──────────────────────────────────────────────────────────┐
│              Consumer Webhook Endpoints                  │
│  https://example.com/webhook                             │
│  (Customer-provided URLs)                                │
└──────────────────────────────────────────────────────────┘
```

### Retry Worker (Background Process)

```python
# retry_worker.py
import time
from datetime import datetime

def retry_worker():
    """Background worker to retry pending webhook deliveries."""
    while True:
        # Find deliveries ready for retry
        deliveries = db.query("""
            SELECT delivery_id
            FROM webhook_deliveries
            WHERE status = 'pending' AND next_retry_at <= NOW()
            ORDER BY next_retry_at ASC
            LIMIT 100
        """)

        for delivery in deliveries:
            dispatcher.deliver_webhook(delivery['delivery_id'])

        # Sleep 5 seconds before next batch
        time.sleep(5)
```

---

## Example Usage

```python
# Example 1: Register webhook
webhook_id = str(uuid4())
db.execute("""
    INSERT INTO webhooks (webhook_id, tenant_id, url, secret, events, filters)
    VALUES (%s, %s, %s, %s, %s, %s)
""", (
    webhook_id,
    "tenant_acme",
    "https://example.com/webhook",
    "webhook_secret_abc123",  # User-provided secret
    ["upload.completed"],
    {"parser_type": "bank_statement"}  # Optional filter
))

# Example 2: Dispatch event
dispatcher = WebhookDispatcher(event_stream, db, queue, http_client)

event_type = "upload.completed"
payload = {
    "upload_id": "upl_abc123",
    "status": "parsed",
    "parser_type": "bank_statement",
    "observation_count": 45,
    "timestamp": "2025-10-25T10:15:23Z"
}

dispatcher.dispatch_event(event_type, payload)
# → Finds 1 webhook (matches event_type and filters)
# → Queues delivery job

# Example 3: Deliver webhook (worker function)
delivery_id = "del_xyz789"
dispatcher.deliver_webhook(delivery_id)
# → HTTP POST to https://example.com/webhook
# → Headers: X-Webhook-Signature: sha256=<hmac>, X-Webhook-Retry: 0
# → Body: {"upload_id": "upl_abc123", ...}
# → Response: 200 OK (success)
# → Mark delivery as success

# Example 4: Retry failed delivery
delivery_id = "del_xyz789"
# First attempt failed with 503
dispatcher.schedule_retry(delivery_id, retry_count=0, status_code=503, response_time_ms=1000)
# → Sets next_retry_at = now + 5 seconds
# → Retry worker picks it up after 5 seconds
# → Delivers again (retry_count=1)

# Example 5: Verify signature (consumer side)
payload = {"upload_id": "upl_abc123", ...}
signature = "sha256=abc123..."
secret = "webhook_secret_abc123"

if dispatcher.verify_signature(payload, signature, secret):
    # Process webhook
    process_upload_completed(payload)
else:
    # Reject webhook (spoofed)
    log_security_incident("Webhook signature mismatch")
```

---

## Domain Applicability

**Universal Across All Domains:**

- **Finance:** Webhook for upload.completed → Notify bank's ERP system
- **Healthcare:** Webhook for observation.created → Trigger EHR update
- **Legal:** Webhook for entity.resolved → Notify contract management system
- **Research:** Webhook for observation.created → Update citation database
- **E-Commerce:** Webhook for reconciliation.completed → Trigger invoice approval
- **HR SaaS:** Webhook for upload.completed → Notify payroll system
- **Insurance:** Webhook for reconciliation.completed → Trigger claim approval workflow

**Pattern:** WebhookDispatcher is domain-agnostic, dispatches events to any consumer endpoint regardless of domain.

---

## Related Primitives

- **EventBus (OL, from 3.1):** Source of system events, WebhookDispatcher subscribes to EventBus
- **AuditLogger (OL, from 5.4):** Logs all webhook deliveries for audit trail
- **APIGateway (OL):** Provides API endpoint for webhook registration (POST /v1/webhooks)
- **RateLimiter (OL, from 5.3):** Used to rate limit webhook deliveries per tenant (prevent abuse)

---

## Testing

```python
# Unit Test: Event dispatching
def test_dispatch_event():
    dispatcher = WebhookDispatcher(event_stream, db, queue, http_client)
    # Register webhook
    register_webhook(tenant_id="tenant_acme", url="https://example.com/webhook", events=["upload.completed"])
    # Dispatch event
    dispatcher.dispatch_event("upload.completed", {"upload_id": "upl_abc123"})
    # Verify delivery queued
    deliveries = db.query("SELECT * FROM webhook_deliveries WHERE event_type = 'upload.completed'")
    assert len(deliveries) == 1

# Unit Test: Signature generation
def test_signature_generation():
    dispatcher = WebhookDispatcher(event_stream, db, queue, http_client)
    payload = {"upload_id": "upl_abc123"}
    secret = "webhook_secret"
    signature = dispatcher.generate_signature(payload, secret)
    assert signature.startswith("sha256=")
    assert len(signature) > 40  # HMAC-SHA256 hex digest

# Unit Test: Signature verification
def test_signature_verification():
    dispatcher = WebhookDispatcher(event_stream, db, queue, http_client)
    payload = {"upload_id": "upl_abc123"}
    secret = "webhook_secret"
    signature = dispatcher.generate_signature(payload, secret)
    assert dispatcher.verify_signature(payload, signature, secret) == True
    assert dispatcher.verify_signature(payload, "sha256=invalid", secret) == False

# Integration Test: End-to-end delivery
def test_end_to_end_delivery():
    # Start mock webhook server
    mock_server = start_mock_webhook_server(port=9000)
    # Register webhook
    register_webhook(url="http://localhost:9000/webhook", events=["upload.completed"])
    # Dispatch event
    dispatcher.dispatch_event("upload.completed", {"upload_id": "upl_abc123"})
    # Wait for delivery
    time.sleep(2)
    # Verify delivery
    deliveries = mock_server.get_deliveries()
    assert len(deliveries) == 1
    assert deliveries[0]['event_type'] == "upload.completed"
    assert "X-Webhook-Signature" in deliveries[0]['headers']

# Integration Test: Retry logic
def test_retry_logic():
    # Start mock server that returns 503 (retriable)
    mock_server = start_failing_webhook_server(status_code=503)
    # Register webhook
    register_webhook(url="http://localhost:9000/webhook", events=["upload.completed"])
    # Dispatch event
    dispatcher.dispatch_event("upload.completed", {"upload_id": "upl_abc123"})
    # Wait for retries
    time.sleep(10)  # 5s + 5s (retry 1 + retry 2)
    # Verify retry count
    delivery = db.query("SELECT retry_count FROM webhook_deliveries WHERE event_type = 'upload.completed'")
    assert delivery['retry_count'] >= 2
```

---

## Performance

- **Event Dispatching:** p95 < 50ms (query webhooks + queue deliveries)
- **Webhook Delivery:** p95 < 10s (includes remote endpoint latency)
- **Signature Generation:** p95 < 5ms (HMAC-SHA256 computation)
- **Throughput:** 500 events/s (with 10 delivery workers)

**Optimization:** Parallel delivery workers (10 workers = 10x throughput).

---

## Metadata

- **Lines of Code:** ~600 (Python implementation)
- **Dependencies:** Event stream (Kafka/Redis), Database, Queue, HTTP client, HMAC library
- **Deployment:** Runs as background service (listener + retry worker)
