# APIKeyManagementPanel (IL Component #55)

**Domain:** Cross-Cutting (API Management)
**Layer:** Interface Layer (IL)
**Vertical:** 5.5 Public API Contracts
**Component Number:** 55
**Created:** 2025-10-25

---

## Purpose

Web UI component for managing API keys: list all keys, generate new keys with scope selection, revoke keys, copy keys to clipboard, and display key usage statistics.

---

## Visual Structure

```
┌─────────────────────────────────────────────────────────────────┐
│  API Keys                                           [+ New Key]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Production Key                          tc_live_abc***    │  │
│  │ Created: 2025-10-01  Last used: 2 hours ago              │  │
│  │ Scopes: read, write                                       │  │
│  │ [Copy Key] [Revoke]                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Test Key (Development)                  tc_test_xyz***    │  │
│  │ Created: 2025-09-15  Last used: never                    │  │
│  │ Scopes: read                                              │  │
│  │ [Copy Key] [Revoke]                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Integration Key (REVOKED)               tc_live_old***    │  │
│  │ Created: 2025-08-01  Revoked: 2025-10-05                 │  │
│  │ Reason: Key leaked on GitHub                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌────────────────────── Generate New API Key ──────────────────────┐
│                                                                    │
│  Key Name: [Production Key v2___________________]                 │
│                                                                    │
│  Scopes:                                                           │
│    ☑ Read (view observations, entities, reconciliations)          │
│    ☑ Write (create uploads, trigger reconciliations)              │
│    ☐ Admin (manage users, access policies, view audit logs)       │
│                                                                    │
│  Expiration: [○ Never  ● Custom Date: 2026-01-01____]             │
│                                                                    │
│  ⚠️  Warning: Copy this key immediately. You won't be able to     │
│     see it again after closing this dialog.                       │
│                                                                    │
│  [Cancel]                                  [Generate Key]          │
└────────────────────────────────────────────────────────────────────┘

┌──────────────────── API Key Generated ───────────────────────────┐
│                                                                    │
│  Your new API key:                                                 │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ tc_live_3K7mP9xQ2jR8vN5wL1tY4uA6bC0dE3K7mP9xQ2jR8       │    │
│  │                                              [Copy] ✓     │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                    │
│  ⚠️  Save this key securely. It won't be shown again.             │
│                                                                    │
│  [Close]                                                           │
└────────────────────────────────────────────────────────────────────┘
```

---

## Props Interface

```typescript
interface APIKeyManagementPanelProps {
  tenantId: string;
  onKeyGenerated?: (apiKey: APIKey) => void;
  onKeyRevoked?: (apiKeyId: string) => void;
}

interface APIKey {
  api_key_id: string;
  key_prefix: string;  // e.g., "tc_live_abc***"
  scopes: string[];  // ['read', 'write', 'admin']
  created_at: string;  // ISO 8601
  last_used_at: string | null;  // ISO 8601 or null
  expires_at: string | null;  // ISO 8601 or null
  revoked: boolean;
  revoked_at: string | null;
  revocation_reason: string | null;
  metadata: {
    name: string;
    created_by: string;
  };
}

interface GenerateKeyRequest {
  name: string;
  scopes: string[];
  expires_at: string | null;
}
```

---

## State Management

```typescript
interface APIKeyManagementState {
  keys: APIKey[];
  loading: boolean;
  showGenerateDialog: boolean;
  showKeyDialog: boolean;
  generatedKey: string | null;  // Full API key (shown once)
  generateForm: {
    name: string;
    scopes: string[];
    expires_at: string | null;
  };
  error: string | null;
}
```

---

## Behavior Specifications

### 1. Load API Keys (on Mount)

**Trigger:** Component mounts

**API Call:**
```http
GET /api/keys?tenant_id=${tenantId}
```

**Success Response:**
```json
{
  "keys": [
    {
      "api_key_id": "key_abc123",
      "key_prefix": "tc_live_abc***",
      "scopes": ["read", "write"],
      "created_at": "2025-10-01T10:00:00Z",
      "last_used_at": "2025-10-25T08:15:23Z",
      "expires_at": null,
      "revoked": false,
      "metadata": {"name": "Production Key", "created_by": "usr_admin"}
    }
  ]
}
```

**UI Update:**
- Display keys in list (sorted by created_at DESC)
- Show revoked keys at bottom (grayed out)

---

### 2. Generate New Key

**Trigger:** User clicks "+ New Key" button

**Dialog Opens:**
- Show "Generate New API Key" dialog
- Form fields: name, scopes (checkboxes), expiration (radio + date picker)

**User Fills Form:**
```typescript
{
  name: "Production Key v2",
  scopes: ["read", "write"],
  expires_at: null  // Never expires
}
```

**User Clicks "Generate Key":**

**API Call:**
```http
POST /api/keys
Content-Type: application/json

{
  "tenant_id": "tenant_acme",
  "name": "Production Key v2",
  "scopes": ["read", "write"],
  "expires_at": null
}
```

**Success Response:**
```json
{
  "api_key_id": "key_xyz789",
  "api_key": "tc_live_3K7mP9xQ2jR8vN5wL1tY4uA6bC0dE3K7mP9xQ2jR8",
  "key_prefix": "tc_live_3K7***",
  "scopes": ["read", "write"],
  "created_at": "2025-10-25T10:15:23Z",
  "expires_at": null,
  "metadata": {"name": "Production Key v2", "created_by": "usr_admin"}
}
```

**UI Update:**
- Close "Generate New API Key" dialog
- Open "API Key Generated" dialog with full api_key
- Show warning "Save this key securely. It won't be shown again."
- Provide [Copy] button (copies to clipboard)
- After user closes dialog, add new key to list (showing only key_prefix)

---

### 3. Copy Key to Clipboard

**Trigger:** User clicks [Copy Key] button (for existing key) OR [Copy] button (for newly generated key)

**Action:**
- Copy key_prefix (existing keys) or full api_key (newly generated) to clipboard
- Show toast notification "Copied to clipboard"
- For existing keys, show warning "Only key prefix available. Full key was shown once at generation."

---

### 4. Revoke Key

**Trigger:** User clicks [Revoke] button

**Confirmation Dialog:**
```
Revoke API Key?

This will immediately invalidate the key "Production Key".
All API requests using this key will be rejected.

Reason for revocation (required):
[Key leaked on GitHub_________________]

[Cancel]  [Revoke Key]
```

**User Clicks "Revoke Key":**

**API Call:**
```http
DELETE /api/keys/${apiKeyId}
Content-Type: application/json

{
  "revocation_reason": "Key leaked on GitHub"
}
```

**Success Response:**
```json
{
  "api_key_id": "key_abc123",
  "revoked": true,
  "revoked_at": "2025-10-25T10:20:00Z",
  "revocation_reason": "Key leaked on GitHub"
}
```

**UI Update:**
- Move revoked key to bottom of list
- Gray out key, show "REVOKED" badge
- Display revocation reason
- Remove [Revoke] button (already revoked)
- Show toast notification "API key revoked"

---

### 5. Display Key Usage Statistics

**For Each Key:**
- **Last Used:** "2 hours ago" (relative time) OR "never" (if last_used_at is null)
- **Created:** "2025-10-01" (absolute date)
- **Expires:** "Never" OR "2026-01-01" (absolute date)

**Tooltip on Hover:**
```
Last used: 2025-10-25 08:15:23 UTC
Created by: usr_admin
Request count (last 30 days): 15,234
```

---

## Accessibility

- **ARIA Labels:**
  - `aria-label="Generate new API key"` on "+ New Key" button
  - `aria-label="Copy API key to clipboard"` on [Copy] buttons
  - `aria-label="Revoke API key"` on [Revoke] buttons

- **Keyboard Navigation:**
  - Tab through keys list
  - Enter to open key details
  - Arrow keys to navigate between keys

- **Screen Reader:**
  - Announce "API key generated" when key created
  - Announce "API key copied to clipboard" when copied
  - Announce "API key revoked" when revoked

---

## Error Handling

| Error Scenario | API Response | UI Behavior |
|----------------|--------------|-------------|
| Network error (loading keys) | Timeout | Show error banner "Failed to load API keys. Retry?" with [Retry] button |
| Invalid scopes (generation) | HTTP 400 "Invalid scope: 'invalid'" | Show inline error "Invalid scope selected" below checkboxes |
| Duplicate key name | HTTP 409 "Key name already exists" | Show inline error "Key name already in use" below name field |
| Revocation reason required | HTTP 400 "Revocation reason required" | Show inline error "Please provide a reason" below reason field |

---

## Domain Applicability

**Universal Across All Domains:**

- **Finance:** Bank admin generates API key for ERP integration
- **Healthcare:** Hospital IT generates API key for EHR system
- **Legal:** Law firm admin generates API key for contract management system
- **Research (RSRCH - Utilitario):** RSRCH team admin generates API key for VC firm CRM integration (querying founder facts)
- **E-Commerce:** SaaS admin generates API key for invoice processing
- **HR SaaS:** HR admin generates API key for payroll system
- **Insurance:** Insurance IT generates API key for claims portal

**Pattern:** APIKeyManagementPanel is domain-agnostic, used by any tenant admin managing API integrations.

---

## Related Components

- **WebhookManagementPanel (IL #56):** Sibling component for managing webhooks
- **APIGateway (OL):** Uses generated API keys for authentication
- **APIKeyValidator (OL):** Validates API keys on each request
- **AuditLogger (OL, from 5.4):** Logs key generation/revocation events

---

## Technical Implementation

```typescript
// APIKeyManagementPanel.tsx
import React, { useState, useEffect } from 'react';

export function APIKeyManagementPanel({ tenantId }: APIKeyManagementPanelProps) {
  const [keys, setKeys] = useState<APIKey[]>([]);
  const [loading, setLoading] = useState(true);
  const [showGenerateDialog, setShowGenerateDialog] = useState(false);
  const [showKeyDialog, setShowKeyDialog] = useState(false);
  const [generatedKey, setGeneratedKey] = useState<string | null>(null);

  // Load keys on mount
  useEffect(() => {
    loadKeys();
  }, [tenantId]);

  async function loadKeys() {
    setLoading(true);
    try {
      const response = await fetch(`/api/keys?tenant_id=${tenantId}`);
      const data = await response.json();
      setKeys(data.keys);
    } catch (error) {
      // Error handling
    } finally {
      setLoading(false);
    }
  }

  async function generateKey(request: GenerateKeyRequest) {
    try {
      const response = await fetch('/api/keys', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ tenant_id: tenantId, ...request })
      });
      const data = await response.json();

      // Show full key dialog
      setGeneratedKey(data.api_key);
      setShowGenerateDialog(false);
      setShowKeyDialog(true);

      // Add to keys list (will show only prefix)
      setKeys([...keys, data]);
    } catch (error) {
      // Error handling
    }
  }

  async function revokeKey(apiKeyId: string, reason: string) {
    try {
      await fetch(`/api/keys/${apiKeyId}`, {
        method: 'DELETE',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ revocation_reason: reason })
      });

      // Update key in list
      setKeys(keys.map(k => k.api_key_id === apiKeyId
        ? { ...k, revoked: true, revoked_at: new Date().toISOString(), revocation_reason: reason }
        : k
      ));
    } catch (error) {
      // Error handling
    }
  }

  function copyToClipboard(text: string) {
    navigator.clipboard.writeText(text);
    // Show toast notification
  }

  return (
    <div className="api-key-management-panel">
      <div className="header">
        <h2>API Keys</h2>
        <button onClick={() => setShowGenerateDialog(true)}>+ New Key</button>
      </div>

      <div className="keys-list">
        {keys.map(key => (
          <APIKeyCard
            key={key.api_key_id}
            apiKey={key}
            onCopy={() => copyToClipboard(key.key_prefix)}
            onRevoke={(reason) => revokeKey(key.api_key_id, reason)}
          />
        ))}
      </div>

      {showGenerateDialog && (
        <GenerateKeyDialog
          onGenerate={generateKey}
          onClose={() => setShowGenerateDialog(false)}
        />
      )}

      {showKeyDialog && generatedKey && (
        <GeneratedKeyDialog
          apiKey={generatedKey}
          onCopy={() => copyToClipboard(generatedKey)}
          onClose={() => setShowKeyDialog(false)}
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
- **Dependencies:** fetch API, clipboard API, date formatting library (date-fns)
- **API Endpoints:** GET /api/keys, POST /api/keys, DELETE /api/keys/:id
