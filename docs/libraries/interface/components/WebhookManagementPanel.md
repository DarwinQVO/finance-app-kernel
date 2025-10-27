# WebhookManagementPanel (IL Component #56)

**Domain:** Cross-Cutting (API Management)
**Layer:** Interface Layer (IL)
**Vertical:** 5.5 Public API Contracts
**Component Number:** 56
**Created:** 2025-10-25

---

## Purpose

Web UI component for managing webhooks: list all webhooks, register new webhooks with event selection and filters, test webhooks, view delivery logs, and resume failed webhooks.

---

## Visual Structure

```
┌─────────────────────────────────────────────────────────────────┐
│  Webhooks                                         [+ New Webhook]│
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Upload Notifications                      ● Active        │  │
│  │ https://example.com/webhook                               │  │
│  │ Events: upload.completed, upload.failed                   │  │
│  │ Last delivery: 5 minutes ago (200 OK, 342ms)             │  │
│  │ [Test] [View Logs] [Delete]                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Entity Updates                            ● Active        │  │
│  │ https://api.partner.com/webhook                           │  │
│  │ Events: entity.resolved, entity.merged                    │  │
│  │ Filters: type=transaction                                 │  │
│  │ Last delivery: 2 hours ago (200 OK, 128ms)               │  │
│  │ [Test] [View Logs] [Delete]                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Reconciliation Alerts                     ⚠️ Failed       │  │
│  │ https://old-endpoint.com/webhook                          │  │
│  │ Events: reconciliation.completed                          │  │
│  │ Last delivery: 1 day ago (503 Service Unavailable)       │  │
│  │ Failed after 5 retries. Reason: Endpoint unreachable     │  │
│  │ [Resume] [View Logs] [Delete]                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌────────────────────── Register New Webhook ──────────────────────┐
│                                                                    │
│  Webhook URL: [https://example.com/webhook____________]           │
│  ⓘ Must be HTTPS. No localhost or private IPs allowed.            │
│                                                                    │
│  Secret (for signature verification):                             │
│  [webhook_secret_abc123________________] [Generate]               │
│  ⓘ Used to generate X-Webhook-Signature header                    │
│                                                                    │
│  Events (select at least one):                                    │
│    ☑ upload.completed                                             │
│    ☐ upload.failed                                                │
│    ☐ observation.created                                          │
│    ☐ entity.resolved                                              │
│    ☐ entity.merged                                                │
│    ☑ reconciliation.completed                                     │
│                                                                    │
│  Filters (optional, JSON):                                        │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ {                                                           │  │
│  │   "parser_type": "bank_statement"                          │  │
│  │ }                                                           │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ⓘ Only trigger webhook when payload matches all filters          │
│                                                                    │
│  [Cancel]                                  [Register Webhook]     │
└────────────────────────────────────────────────────────────────────┘

┌──────────────────── Webhook Delivery Logs ───────────────────────┐
│  Upload Notifications (https://example.com/webhook)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 2025-10-25 10:15:23 UTC  upload.completed  ✓ 200 OK      │   │
│  │ Retry: 0  Response time: 342ms                           │   │
│  │ Payload: {"upload_id": "upl_abc123", "status": "parsed"} │   │
│  │ [View Full Payload]                                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 2025-10-25 09:45:12 UTC  upload.completed  ✓ 200 OK      │   │
│  │ Retry: 0  Response time: 256ms                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 2025-10-25 08:30:45 UTC  upload.failed     ✓ 200 OK      │   │
│  │ Retry: 0  Response time: 189ms                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 2025-10-24 16:20:11 UTC  upload.completed  ❌ 503         │   │
│  │ Retry: 5  Response time: 10000ms (timeout)               │   │
│  │ Error: Service Unavailable                               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                    │
│  [Load More]                                                       │
└────────────────────────────────────────────────────────────────────┘
```

---

## Props Interface

```typescript
interface WebhookManagementPanelProps {
  tenantId: string;
  onWebhookRegistered?: (webhook: Webhook) => void;
  onWebhookDeleted?: (webhookId: string) => void;
}

interface Webhook {
  webhook_id: string;
  tenant_id: string;
  url: string;
  secret: string;  // Never shown in UI (for security)
  events: string[];  // ['upload.completed', 'observation.created']
  filters: Record<string, any> | null;  // {"parser_type": "bank_statement"}
  status: 'active' | 'failed' | 'disabled';
  last_delivery_at: string | null;  // ISO 8601
  last_delivery_status: number | null;  // HTTP status code
  last_delivery_response_time_ms: number | null;
  failure_count: number;
  created_at: string;  // ISO 8601
}

interface WebhookDelivery {
  delivery_id: string;
  webhook_id: string;
  event_type: string;
  payload: Record<string, any>;
  signature: string;  // HMAC-SHA256 signature
  status: 'pending' | 'success' | 'failed';
  status_code: number | null;
  response_time_ms: number | null;
  retry_count: number;
  delivered_at: string | null;  // ISO 8601
  created_at: string;  // ISO 8601
}

interface RegisterWebhookRequest {
  url: string;
  secret: string;
  events: string[];
  filters: Record<string, any> | null;
}
```

---

## State Management

```typescript
interface WebhookManagementState {
  webhooks: Webhook[];
  loading: boolean;
  showRegisterDialog: boolean;
  showLogsDialog: boolean;
  selectedWebhook: Webhook | null;
  deliveryLogs: WebhookDelivery[];
  registerForm: {
    url: string;
    secret: string;
    events: string[];
    filters: string;  // JSON string
  };
  error: string | null;
}
```

---

## Behavior Specifications

### 1. Load Webhooks (on Mount)

**Trigger:** Component mounts

**API Call:**
```http
GET /api/webhooks?tenant_id=${tenantId}
```

**Success Response:**
```json
{
  "webhooks": [
    {
      "webhook_id": "wh_abc123",
      "tenant_id": "tenant_acme",
      "url": "https://example.com/webhook",
      "events": ["upload.completed", "upload.failed"],
      "filters": null,
      "status": "active",
      "last_delivery_at": "2025-10-25T10:10:00Z",
      "last_delivery_status": 200,
      "last_delivery_response_time_ms": 342,
      "failure_count": 0,
      "created_at": "2025-10-01T10:00:00Z"
    }
  ]
}
```

**UI Update:**
- Display webhooks in list (sorted by created_at DESC)
- Show status badge (● Active, ⚠️ Failed, ○ Disabled)
- Show last delivery info (relative time, status code, response time)

---

### 2. Register New Webhook

**Trigger:** User clicks "+ New Webhook" button

**Dialog Opens:**
- Show "Register New Webhook" dialog
- Form fields: url, secret (with [Generate] button), events (checkboxes), filters (JSON textarea)

**User Fills Form:**
```typescript
{
  url: "https://example.com/webhook",
  secret: "webhook_secret_abc123",  // User types OR clicks [Generate]
  events: ["upload.completed", "reconciliation.completed"],
  filters: '{"parser_type": "bank_statement"}'  // JSON string
}
```

**User Clicks "Register Webhook":**

**API Call:**
```http
POST /api/webhooks
Content-Type: application/json

{
  "tenant_id": "tenant_acme",
  "url": "https://example.com/webhook",
  "secret": "webhook_secret_abc123",
  "events": ["upload.completed", "reconciliation.completed"],
  "filters": {"parser_type": "bank_statement"}
}
```

**Success Response:**
```json
{
  "webhook_id": "wh_xyz789",
  "tenant_id": "tenant_acme",
  "url": "https://example.com/webhook",
  "events": ["upload.completed", "reconciliation.completed"],
  "filters": {"parser_type": "bank_statement"},
  "status": "active",
  "created_at": "2025-10-25T10:15:23Z"
}
```

**UI Update:**
- Close "Register New Webhook" dialog
- Add new webhook to list
- Show toast notification "Webhook registered"

---

### 3. Test Webhook

**Trigger:** User clicks [Test] button

**API Call:**
```http
POST /api/webhooks/${webhookId}/test
```

**Backend Behavior:**
- Send test event to webhook URL
- Use dummy payload: `{"event": "test", "webhook_id": "wh_abc123", "timestamp": "2025-10-25T10:15:23Z"}`
- Wait for response (10 second timeout)

**Success Response:**
```json
{
  "status": "success",
  "status_code": 200,
  "response_time_ms": 245,
  "response_body": "OK"
}
```

**UI Update:**
- Show toast notification "Test successful (200 OK, 245ms)"

**Error Response:**
```json
{
  "status": "failed",
  "status_code": 503,
  "error": "Service Unavailable"
}
```

**UI Update:**
- Show error toast "Test failed (503 Service Unavailable)"

---

### 4. View Delivery Logs

**Trigger:** User clicks [View Logs] button

**API Call:**
```http
GET /api/webhooks/${webhookId}/deliveries?limit=50
```

**Success Response:**
```json
{
  "deliveries": [
    {
      "delivery_id": "del_1",
      "webhook_id": "wh_abc123",
      "event_type": "upload.completed",
      "payload": {"upload_id": "upl_abc123", "status": "parsed"},
      "signature": "sha256=abc123...",
      "status": "success",
      "status_code": 200,
      "response_time_ms": 342,
      "retry_count": 0,
      "delivered_at": "2025-10-25T10:15:23Z",
      "created_at": "2025-10-25T10:15:23Z"
    }
  ]
}
```

**UI Update:**
- Open "Webhook Delivery Logs" dialog
- Display deliveries in reverse chronological order
- Show status icon (✓ success, ❌ failed, ⏳ pending)
- Show retry count, response time, status code
- Provide [View Full Payload] button (expands payload JSON)

---

### 5. Resume Failed Webhook

**Trigger:** User clicks [Resume] button (only shown for failed webhooks)

**API Call:**
```http
POST /api/webhooks/${webhookId}/resume
```

**Backend Behavior:**
- Set status = 'active', failure_count = 0
- Retry pending deliveries (if any)

**Success Response:**
```json
{
  "webhook_id": "wh_abc123",
  "status": "active",
  "failure_count": 0
}
```

**UI Update:**
- Update webhook status badge to ● Active
- Show toast notification "Webhook resumed"

---

### 6. Delete Webhook

**Trigger:** User clicks [Delete] button

**Confirmation Dialog:**
```
Delete Webhook?

This will permanently delete the webhook "Upload Notifications".
All future events will not be delivered to this endpoint.

[Cancel]  [Delete Webhook]
```

**User Clicks "Delete Webhook":**

**API Call:**
```http
DELETE /api/webhooks/${webhookId}
```

**Success Response:**
```json
{
  "webhook_id": "wh_abc123",
  "deleted": true
}
```

**UI Update:**
- Remove webhook from list
- Show toast notification "Webhook deleted"

---

## Accessibility

- **ARIA Labels:**
  - `aria-label="Register new webhook"` on "+ New Webhook" button
  - `aria-label="Test webhook"` on [Test] buttons
  - `aria-label="View delivery logs"` on [View Logs] buttons
  - `aria-label="Delete webhook"` on [Delete] buttons

- **Keyboard Navigation:**
  - Tab through webhooks list
  - Enter to open webhook details
  - Arrow keys to navigate between webhooks

- **Screen Reader:**
  - Announce "Webhook registered" when created
  - Announce "Test successful" or "Test failed" after test
  - Announce "Webhook deleted" when deleted

---

## Error Handling

| Error Scenario | API Response | UI Behavior |
|----------------|--------------|-------------|
| Network error (loading webhooks) | Timeout | Show error banner "Failed to load webhooks. Retry?" with [Retry] button |
| Invalid URL (localhost) | HTTP 400 "URL must be HTTPS" | Show inline error "Invalid URL. Must be HTTPS, no localhost." below URL field |
| Invalid JSON filters | HTTP 400 "Invalid JSON" | Show inline error "Invalid JSON syntax" below filters field |
| No events selected | HTTP 400 "At least one event required" | Show inline error "Select at least one event" below checkboxes |
| Test webhook timeout | HTTP 408 "Request timeout" | Show toast "Test failed (timeout after 10s)" |
| Delete failed (webhook has pending deliveries) | HTTP 409 "Webhook has pending deliveries" | Show toast "Cannot delete webhook with pending deliveries. Resume or wait for retries to complete." |

---

## Domain Applicability

**Universal Across All Domains:**

- **Finance:** Bank registers webhook for upload.completed → Notify ERP
- **Healthcare:** Hospital registers webhook for observation.created → Update EHR
- **Legal:** Law firm registers webhook for entity.resolved → Notify contract system
- **Research (RSRCH - Utilitario):** RSRCH team registers webhook for fact.discovered → Update VC firm CRM with founder facts
- **E-Commerce:** SaaS registers webhook for reconciliation.completed → Trigger invoice approval
- **HR SaaS:** HR platform registers webhook for upload.completed → Notify payroll
- **Insurance:** Insurance company registers webhook for reconciliation.completed → Trigger claim approval

**Pattern:** WebhookManagementPanel is domain-agnostic, used by any tenant admin managing real-time integrations.

---

## Related Components

- **APIKeyManagementPanel (IL #55):** Sibling component for managing API keys
- **WebhookDispatcher (OL):** Dispatches events to registered webhooks
- **APIGateway (OL):** Provides API endpoints for webhook registration/testing
- **AuditLogger (OL, from 5.4):** Logs webhook registration/deletion events

---

## Technical Implementation

```typescript
// WebhookManagementPanel.tsx
import React, { useState, useEffect } from 'react';

export function WebhookManagementPanel({ tenantId }: WebhookManagementPanelProps) {
  const [webhooks, setWebhooks] = useState<Webhook[]>([]);
  const [loading, setLoading] = useState(true);
  const [showRegisterDialog, setShowRegisterDialog] = useState(false);
  const [showLogsDialog, setShowLogsDialog] = useState(false);
  const [selectedWebhook, setSelectedWebhook] = useState<Webhook | null>(null);
  const [deliveryLogs, setDeliveryLogs] = useState<WebhookDelivery[]>([]);

  // Load webhooks on mount
  useEffect(() => {
    loadWebhooks();
  }, [tenantId]);

  async function loadWebhooks() {
    setLoading(true);
    try {
      const response = await fetch(`/api/webhooks?tenant_id=${tenantId}`);
      const data = await response.json();
      setWebhooks(data.webhooks);
    } catch (error) {
      // Error handling
    } finally {
      setLoading(false);
    }
  }

  async function registerWebhook(request: RegisterWebhookRequest) {
    try {
      const response = await fetch('/api/webhooks', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ tenant_id: tenantId, ...request })
      });
      const data = await response.json();

      // Add to webhooks list
      setWebhooks([...webhooks, data]);
      setShowRegisterDialog(false);
      // Show toast notification
    } catch (error) {
      // Error handling
    }
  }

  async function testWebhook(webhookId: string) {
    try {
      const response = await fetch(`/api/webhooks/${webhookId}/test`, {
        method: 'POST'
      });
      const data = await response.json();
      // Show toast notification (success or failed)
    } catch (error) {
      // Error handling
    }
  }

  async function viewDeliveryLogs(webhook: Webhook) {
    try {
      const response = await fetch(`/api/webhooks/${webhook.webhook_id}/deliveries?limit=50`);
      const data = await response.json();
      setDeliveryLogs(data.deliveries);
      setSelectedWebhook(webhook);
      setShowLogsDialog(true);
    } catch (error) {
      // Error handling
    }
  }

  async function resumeWebhook(webhookId: string) {
    try {
      await fetch(`/api/webhooks/${webhookId}/resume`, {
        method: 'POST'
      });

      // Update webhook in list
      setWebhooks(webhooks.map(w => w.webhook_id === webhookId
        ? { ...w, status: 'active', failure_count: 0 }
        : w
      ));
    } catch (error) {
      // Error handling
    }
  }

  async function deleteWebhook(webhookId: string) {
    try {
      await fetch(`/api/webhooks/${webhookId}`, {
        method: 'DELETE'
      });

      // Remove from list
      setWebhooks(webhooks.filter(w => w.webhook_id !== webhookId));
    } catch (error) {
      // Error handling
    }
  }

  return (
    <div className="webhook-management-panel">
      <div className="header">
        <h2>Webhooks</h2>
        <button onClick={() => setShowRegisterDialog(true)}>+ New Webhook</button>
      </div>

      <div className="webhooks-list">
        {webhooks.map(webhook => (
          <WebhookCard
            key={webhook.webhook_id}
            webhook={webhook}
            onTest={() => testWebhook(webhook.webhook_id)}
            onViewLogs={() => viewDeliveryLogs(webhook)}
            onResume={() => resumeWebhook(webhook.webhook_id)}
            onDelete={() => deleteWebhook(webhook.webhook_id)}
          />
        ))}
      </div>

      {showRegisterDialog && (
        <RegisterWebhookDialog
          onRegister={registerWebhook}
          onClose={() => setShowRegisterDialog(false)}
        />
      )}

      {showLogsDialog && selectedWebhook && (
        <DeliveryLogsDialog
          webhook={selectedWebhook}
          deliveries={deliveryLogs}
          onClose={() => setShowLogsDialog(false)}
        />
      )}
    </div>
  );
}
```

---

## Metadata

- **Component Type:** Full-page panel (dashboard section)
- **Framework:** React + TypeScript
- **Dependencies:** fetch API, JSON validator, date formatting library (date-fns)
- **API Endpoints:** GET /api/webhooks, POST /api/webhooks, POST /api/webhooks/:id/test, GET /api/webhooks/:id/deliveries, POST /api/webhooks/:id/resume, DELETE /api/webhooks/:id
