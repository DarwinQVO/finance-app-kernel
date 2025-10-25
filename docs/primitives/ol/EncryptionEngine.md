# EncryptionEngine OL Primitive

**Domain:** Security & Access (Vertical 5.4)
**Layer:** Objective Layer (OL)
**Version:** 1.0.0
**Status:** Specification

---

## Table of Contents

1. [Overview](#overview)
2. [Purpose & Scope](#purpose--scope)
3. [Multi-Domain Applicability](#multi-domain-applicability)
4. [Core Concepts](#core-concepts)
5. [Interface Definition](#interface-definition)
6. [Data Model](#data-model)
7. [Core Functionality](#core-functionality)
8. [Envelope Encryption](#envelope-encryption)
9. [Key Management](#key-management)
10. [Key Rotation](#key-rotation)
11. [Edge Cases](#edge-cases)
12. [Performance Characteristics](#performance-characteristics)
13. [Implementation Notes](#implementation-notes)
14. [Security Considerations](#security-considerations)
15. [Integration Patterns](#integration-patterns)
16. [Multi-Domain Examples](#multi-domain-examples)
17. [Testing Strategy](#testing-strategy)
18. [Configuration](#configuration)
19. [Observability](#observability)
20. [Future Enhancements](#future-enhancements)

---

## Overview

The **EncryptionEngine** provides transparent encryption at rest for sensitive documents and observations. It implements envelope encryption with AWS KMS/GCP Cloud KMS/Azure Key Vault integration, automatic key rotation, and <10ms p95 latency for 5MB documents.

### Key Capabilities

- **Envelope Encryption**: Master Key → DEK → Data (two-layer encryption)
- **AES-256-GCM**: Industry-standard authenticated encryption
- **KMS Integration**: AWS KMS, GCP Cloud KMS, Azure Key Vault support
- **Automatic Key Rotation**: 90-day rotation with zero downtime
- **High Performance**: <10ms p95 for 5MB document encryption/decryption
- **Transparent**: No application changes required
- **Audit Trail**: All encryption operations logged

### Design Philosophy

The EncryptionEngine follows four core principles:

1. **Defense in Depth**: Multiple encryption layers (transport + rest)
2. **Key Separation**: Data keys separate from master keys
3. **Zero Downtime Rotation**: Rotate keys without service interruption
4. **Performance**: <10ms p95 for 5MB documents

---

## Purpose & Scope

### Problem Statement

In document processing systems, encryption at rest poses critical challenges:

- **Regulatory Requirements**: HIPAA, PCI DSS, SOX require encryption at rest
- **Key Management Complexity**: Generating, storing, rotating keys is error-prone
- **Performance Impact**: Naive encryption adds 50-100ms per operation
- **Key Rotation Risk**: Manual rotation causes downtime and data loss
- **No Audit Trail**: Can't prove data was encrypted at specific time
- **Inconsistent Encryption**: Some data encrypted, some not

Traditional approaches have significant limitations:

**Approach 1: Database-Level Encryption**
- ❌ All-or-nothing (entire database encrypted)
- ❌ Can't encrypt specific fields
- ❌ Performance impact on all queries

**Approach 2: Application-Level Encryption**
- ❌ Scattered throughout codebase
- ❌ Inconsistent key management
- ❌ High latency (50-100ms per operation)

**Approach 3: EncryptionEngine Primitive**
- ✅ Envelope encryption for performance
- ✅ <10ms p95 for 5MB documents
- ✅ Automatic key rotation (90 days)
- ✅ KMS integration for secure key storage
- ✅ Full audit trail

### Solution

The EncryptionEngine implements **envelope encryption** with:

1. **Master Key**: Stored in KMS (AWS/GCP/Azure), never exposed
2. **Data Encryption Keys (DEK)**: Generated per document/batch, encrypted with Master Key
3. **AES-256-GCM**: Fast, secure, authenticated encryption
4. **Key Rotation**: Automatic 90-day rotation with graceful migration
5. **Audit Log**: All encrypt/decrypt operations logged

---

## Multi-Domain Applicability

The EncryptionEngine is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Encrypt bank statements, transaction data, account numbers.

**Encryption Requirements**:
- PCI DSS: Credit card data must be encrypted at rest
- SOX: Financial reports encrypted for auditability
- Encryption algorithm: AES-256-GCM
- Key rotation: 90 days

**Example**:
```typescript
// Encrypt bank statement before storage
const statement = {
  account_number: "123456789012",
  transactions: [...],
  balance: 50000.00
};

const encrypted = await encryptionEngine.encryptDocument({
  document: JSON.stringify(statement),
  document_id: "statement_123",
  metadata: {
    type: "bank_statement",
    account: "123456789012"
  }
});

// Store encrypted data
await documentStore.save({
  document_id: "statement_123",
  encrypted_data: encrypted.ciphertext,
  encrypted_dek: encrypted.encrypted_dek,
  iv: encrypted.iv,
  auth_tag: encrypted.auth_tag
});

// Later: decrypt for authorized user
const decrypted = await encryptionEngine.decryptDocument({
  ciphertext: encrypted_data,
  encrypted_dek: encrypted.encrypted_dek,
  iv: encrypted.iv,
  auth_tag: encrypted.auth_tag,
  document_id: "statement_123"
});

const statement = JSON.parse(decrypted.plaintext);
```

**Performance Target**: <10ms p95 for typical statement (1-5MB).

---

### 2. Healthcare

**Use Case**: Encrypt patient records, medical history, test results (PHI).

**Encryption Requirements**:
- HIPAA: PHI must be encrypted at rest and in transit
- Encryption algorithm: AES-256-GCM
- Key rotation: 90 days
- Audit trail: Log all PHI access

**Example**:
```typescript
// Encrypt patient record
const patientRecord = {
  patient_id: "pt_456",
  name: "John Doe",
  ssn: "123-45-6789",
  medical_history: [...],
  test_results: [...]
};

const encrypted = await encryptionEngine.encryptDocument({
  document: JSON.stringify(patientRecord),
  document_id: "patient_pt_456",
  metadata: {
    type: "patient_record",
    patient_id: "pt_456"
  }
});

// Store in database
await patientStore.save({
  patient_id: "pt_456",
  encrypted_data: encrypted.ciphertext,
  encrypted_dek: encrypted.encrypted_dek,
  iv: encrypted.iv,
  auth_tag: encrypted.auth_tag
});
```

**Performance Target**: <10ms p95 for patient record (1-10MB).

---

### 3. Legal

**Use Case**: Encrypt case files, depositions, attorney-client privileged documents.

**Encryption Requirements**:
- Attorney-client privilege: Must prove documents were encrypted
- Audit trail: Log all access for ethical compliance
- Encryption algorithm: AES-256-GCM
- Key rotation: 90 days

**Example**:
```typescript
// Encrypt privileged document
const privilegedDoc = {
  case_id: "case_789",
  document_type: "attorney_client_memo",
  content: "...",
  parties: ["Attorney Jane Smith", "Client John Doe"]
};

const encrypted = await encryptionEngine.encryptDocument({
  document: JSON.stringify(privilegedDoc),
  document_id: "doc_789",
  metadata: {
    type: "privileged",
    case_id: "case_789"
  }
});

// Audit log automatically created
```

**Performance Target**: <15ms p95 for large documents (10-50MB).

---

### 4. HR (Human Resources)

**Use Case**: Encrypt employee SSNs, salaries, performance reviews.

**Encryption Requirements**:
- GDPR: Personal data must be encrypted
- Encryption algorithm: AES-256-GCM
- Key rotation: 90 days
- Data minimization: Only encrypt sensitive fields

**Example**:
```typescript
// Encrypt employee record
const employeeRecord = {
  employee_id: "emp_123",
  name: "Jane Smith",
  ssn: "987-65-4321",
  salary: 125000,
  performance_reviews: [...]
};

const encrypted = await encryptionEngine.encryptDocument({
  document: JSON.stringify(employeeRecord),
  document_id: "employee_emp_123",
  metadata: {
    type: "employee_record",
    employee_id: "emp_123"
  }
});
```

**Performance Target**: <5ms p95 for employee record (<1MB).

---

### 5. E-commerce

**Use Case**: Encrypt customer payment info, shipping addresses, order history.

**Encryption Requirements**:
- PCI DSS: Credit card data must be encrypted
- Encryption algorithm: AES-256-GCM
- Key rotation: 90 days
- Tokenization: Credit cards tokenized + encrypted

**Example**:
```typescript
// Encrypt order with payment info
const order = {
  order_id: "order_456",
  customer_id: "cust_789",
  payment_method: {
    type: "credit_card",
    last_four: "1111",
    encrypted_full_number: "..." // Already tokenized
  },
  shipping_address: {
    street: "123 Main St",
    city: "Anytown",
    zip: "12345"
  }
};

const encrypted = await encryptionEngine.encryptDocument({
  document: JSON.stringify(order),
  document_id: "order_456",
  metadata: {
    type: "order",
    customer_id: "cust_789"
  }
});
```

**Performance Target**: <5ms p95 for order (<500KB).

---

### 6. SaaS

**Use Case**: Encrypt API keys, webhook secrets, customer data.

**Encryption Requirements**:
- SOC 2: Customer data must be encrypted at rest
- Encryption algorithm: AES-256-GCM
- Key rotation: 90 days
- Multi-tenancy: Separate keys per tenant (optional)

**Example**:
```typescript
// Encrypt API key
const apiKey = {
  key_id: "key_123",
  key_value: "tc_live_abc123def456",
  tenant_id: "tenant_abc",
  scopes: ["read", "write"]
};

const encrypted = await encryptionEngine.encryptDocument({
  document: JSON.stringify(apiKey),
  document_id: "api_key_123",
  metadata: {
    type: "api_key",
    tenant_id: "tenant_abc"
  }
});
```

**Performance Target**: <3ms p95 for small secrets (<1KB).

---

### 7. Government

**Use Case**: Encrypt classified documents, citizen records.

**Encryption Requirements**:
- FISMA: Sensitive data must be encrypted
- Classification levels: Encrypt per classification (Top Secret, Secret, Confidential)
- Encryption algorithm: AES-256-GCM
- Key rotation: 90 days (configurable per classification)

**Example**:
```typescript
// Encrypt classified document
const classifiedDoc = {
  document_id: "doc_classified_456",
  classification: "SECRET",
  content: "...",
  originator: "Agency XYZ"
};

const encrypted = await encryptionEngine.encryptDocument({
  document: JSON.stringify(classifiedDoc),
  document_id: "doc_classified_456",
  metadata: {
    type: "classified",
    classification: "SECRET"
  }
});
```

**Performance Target**: <10ms p95 for classified documents (1-10MB).

---

### Cross-Domain Benefits

**Consistent Encryption**: All domains benefit from:
- Standardized encryption at rest
- Automatic key rotation
- Full audit trail
- High performance (<10ms p95)

**Regulatory Compliance**: Supports:
- HIPAA (Healthcare): PHI encryption
- PCI DSS (Payments): Credit card encryption
- GDPR (Privacy): Personal data encryption
- SOX (Finance): Financial data encryption
- FISMA (Government): Sensitive data encryption

---

## Core Concepts

### Envelope Encryption

**Two-Layer Encryption**:

1. **Data Encryption Key (DEK)**:
   - 256-bit random key generated per document/batch
   - Used to encrypt actual data with AES-256-GCM
   - Encrypted with Master Key before storage

2. **Master Key**:
   - Stored in KMS (AWS/GCP/Azure)
   - Never exposed to application
   - Used only to encrypt/decrypt DEKs

**Benefits**:
- **Performance**: Encrypt data locally with DEK, avoid KMS call per document
- **Security**: Master Key never leaves KMS
- **Key Rotation**: Rotate Master Key without re-encrypting all data

### Encryption Flow

**Encryption**:
```
1. Generate random 256-bit DEK
2. Encrypt document with DEK using AES-256-GCM
3. Encrypt DEK with Master Key using KMS
4. Store: encrypted_document + encrypted_dek + iv + auth_tag
```

**Decryption**:
```
1. Retrieve encrypted_dek, encrypted_document, iv, auth_tag
2. Decrypt DEK with Master Key using KMS
3. Decrypt document with DEK using AES-256-GCM
4. Return plaintext document
```

### Key Hierarchy

```
Master Key (KMS)
├── DEK_1 → Document_1
├── DEK_2 → Document_2
├── DEK_3 → Batch (Documents 3-102)
└── ...
```

---

## Interface Definition

### TypeScript Interface

```typescript
interface EncryptionEngine {
  /**
   * Encrypt document using envelope encryption
   *
   * @param request - Encryption request
   * @returns Encrypted document with metadata
   * @throws EncryptionError if encryption fails
   */
  encryptDocument(request: EncryptRequest): Promise<EncryptedDocument>;

  /**
   * Decrypt document using envelope encryption
   *
   * @param request - Decryption request
   * @returns Decrypted document
   * @throws DecryptionError if decryption fails
   */
  decryptDocument(request: DecryptRequest): Promise<DecryptedDocument>;

  /**
   * Rotate Master Key
   *
   * @param options - Rotation options
   * @returns Rotation result
   */
  rotateMasterKey(options?: RotationOptions): Promise<RotationResult>;

  /**
   * Generate new Data Encryption Key (DEK)
   *
   * @returns Generated DEK (plaintext)
   */
  generateDEK(): Promise<Buffer>;

  /**
   * Encrypt DEK with Master Key
   *
   * @param dek - Plaintext DEK
   * @returns Encrypted DEK
   */
  encryptDEK(dek: Buffer): Promise<EncryptedDEK>;

  /**
   * Decrypt DEK with Master Key
   *
   * @param encrypted_dek - Encrypted DEK
   * @returns Plaintext DEK
   */
  decryptDEK(encrypted_dek: EncryptedDEK): Promise<Buffer>;

  /**
   * Get encryption statistics
   *
   * @returns Encryption stats
   */
  getStats(): EncryptionStats;
}
```

---

## Data Model

### EncryptRequest Type

```typescript
interface EncryptRequest {
  document: string | Buffer;      // Document to encrypt (plaintext)
  document_id: string;            // Document identifier
  metadata?: Record<string, any>; // Additional metadata
  use_batch_dek?: boolean;        // Use shared DEK for batch (optional)
}
```

### EncryptedDocument Type

```typescript
interface EncryptedDocument {
  // Encrypted Data
  ciphertext: Buffer;             // Encrypted document
  encrypted_dek: EncryptedDEK;    // Encrypted DEK
  iv: Buffer;                     // Initialization vector (12 bytes for GCM)
  auth_tag: Buffer;               // Authentication tag (16 bytes)

  // Metadata
  document_id: string;
  algorithm: string;              // "AES-256-GCM"
  key_version: string;            // Master Key version used
  encrypted_at: string;           // ISO timestamp
}
```

### DecryptRequest Type

```typescript
interface DecryptRequest {
  ciphertext: Buffer;             // Encrypted document
  encrypted_dek: EncryptedDEK;    // Encrypted DEK
  iv: Buffer;                     // Initialization vector
  auth_tag: Buffer;               // Authentication tag
  document_id: string;            // Document identifier
}
```

### DecryptedDocument Type

```typescript
interface DecryptedDocument {
  plaintext: string | Buffer;     // Decrypted document
  document_id: string;
  decrypted_at: string;           // ISO timestamp
  key_version: string;            // Master Key version used
}
```

### EncryptedDEK Type

```typescript
interface EncryptedDEK {
  ciphertext_blob: Buffer;        // KMS-encrypted DEK
  key_id: string;                 // KMS key ID/ARN
}
```

### RotationOptions Type

```typescript
interface RotationOptions {
  force?: boolean;                // Force rotation even if not due
  new_key_id?: string;            // Specify new KMS key (optional)
}
```

### RotationResult Type

```typescript
interface RotationResult {
  old_key_id: string;
  new_key_id: string;
  rotated_at: string;             // ISO timestamp
  documents_reencrypted: number;  // Count of re-encrypted documents
}
```

### EncryptionStats Type

```typescript
interface EncryptionStats {
  // Throughput
  documents_encrypted_total: number;
  documents_decrypted_total: number;
  encryption_per_second: number;
  decryption_per_second: number;

  // Performance
  encryption_latency_p50_ms: number;
  encryption_latency_p95_ms: number;
  encryption_latency_p99_ms: number;
  decryption_latency_p50_ms: number;
  decryption_latency_p95_ms: number;
  decryption_latency_p99_ms: number;

  // Keys
  current_key_version: string;
  key_last_rotated_at: string;
  days_since_rotation: number;

  // Errors
  encryption_errors_total: number;
  decryption_errors_total: number;
}
```

### Database Schema (PostgreSQL)

```sql
-- Encrypted documents metadata table
CREATE TABLE encrypted_documents (
  -- Primary Key
  document_id VARCHAR(128) PRIMARY KEY,

  -- Encrypted Data (stored separately in S3/blob storage)
  ciphertext_location VARCHAR(512) NOT NULL,

  -- Encryption Metadata
  encrypted_dek BYTEA NOT NULL,
  kms_key_id VARCHAR(256) NOT NULL,
  iv BYTEA NOT NULL,
  auth_tag BYTEA NOT NULL,
  algorithm VARCHAR(32) NOT NULL DEFAULT 'AES-256-GCM',

  -- Timestamps
  encrypted_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Key rotation history table
CREATE TABLE key_rotation_history (
  -- Primary Key
  rotation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Key Info
  old_key_id VARCHAR(256) NOT NULL,
  new_key_id VARCHAR(256) NOT NULL,

  -- Rotation Metadata
  rotated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  documents_reencrypted INTEGER NOT NULL DEFAULT 0,
  rotation_duration_ms INTEGER
);

-- Encryption audit log
CREATE TABLE encryption_audit_log (
  -- Primary Key
  log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Operation
  operation VARCHAR(32) NOT NULL, -- "encrypt" | "decrypt"
  document_id VARCHAR(128) NOT NULL,

  -- User Context
  user_id VARCHAR(128),
  tenant_id VARCHAR(128),

  -- Metadata
  key_version VARCHAR(256),
  success BOOLEAN NOT NULL,
  error_message TEXT,

  -- Timestamp
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_encrypted_documents_encrypted_at ON encrypted_documents(encrypted_at DESC);
CREATE INDEX idx_key_rotation_history_rotated_at ON key_rotation_history(rotated_at DESC);
CREATE INDEX idx_encryption_audit_log_document_id ON encryption_audit_log(document_id);
CREATE INDEX idx_encryption_audit_log_user_id ON encryption_audit_log(user_id);
CREATE INDEX idx_encryption_audit_log_created_at ON encryption_audit_log(created_at DESC);
```

---

## Core Functionality

### 1. encryptDocument()

Encrypt document using envelope encryption.

#### Signature

```typescript
encryptDocument(request: EncryptRequest): Promise<EncryptedDocument>
```

#### Behavior

1. **Generate DEK**:
   - Generate random 256-bit key

2. **Encrypt Document**:
   - Encrypt with AES-256-GCM using DEK
   - Generate random 12-byte IV
   - Get 16-byte authentication tag

3. **Encrypt DEK**:
   - Encrypt DEK with Master Key via KMS

4. **Log**:
   - Insert into `encryption_audit_log`

5. **Return**:
   - Return encrypted document with metadata

#### Algorithm

```typescript
async encryptDocument(request: EncryptRequest): Promise<EncryptedDocument> {
  // 1. Generate DEK
  const dek = await this.generateDEK();

  // 2. Encrypt document with DEK
  const iv = randomBytes(12); // 12 bytes for GCM
  const cipher = createCipheriv('aes-256-gcm', dek, iv);

  const documentBuffer = Buffer.isBuffer(request.document)
    ? request.document
    : Buffer.from(request.document, 'utf8');

  let ciphertext = cipher.update(documentBuffer);
  ciphertext = Buffer.concat([ciphertext, cipher.final()]);

  const auth_tag = cipher.getAuthTag(); // 16 bytes

  // 3. Encrypt DEK with Master Key
  const encrypted_dek = await this.encryptDEK(dek);

  // 4. Log
  await this.logEncryption({
    operation: 'encrypt',
    document_id: request.document_id,
    key_version: encrypted_dek.key_id,
    success: true
  });

  // 5. Return
  return {
    ciphertext,
    encrypted_dek,
    iv,
    auth_tag,
    document_id: request.document_id,
    algorithm: 'AES-256-GCM',
    key_version: encrypted_dek.key_id,
    encrypted_at: new Date().toISOString()
  };
}
```

#### Example

```typescript
const encrypted = await encryptionEngine.encryptDocument({
  document: "Sensitive medical record...",
  document_id: "patient_record_123"
});

console.log(`Encrypted with key: ${encrypted.key_version}`);
console.log(`Ciphertext size: ${encrypted.ciphertext.length} bytes`);

// Store in database
await db.query(`
  INSERT INTO encrypted_documents (document_id, encrypted_dek, iv, auth_tag, kms_key_id)
  VALUES ($1, $2, $3, $4, $5)
`, [
  encrypted.document_id,
  encrypted.encrypted_dek.ciphertext_blob,
  encrypted.iv,
  encrypted.auth_tag,
  encrypted.encrypted_dek.key_id
]);

// Store ciphertext in blob storage
await s3.putObject({
  Bucket: 'encrypted-documents',
  Key: encrypted.document_id,
  Body: encrypted.ciphertext
});
```

#### Performance

- **Target Latency**: <10ms p95 for 5MB document
- **Breakdown**:
  - DEK generation: <1ms
  - AES-256-GCM encryption: <5ms (5MB)
  - KMS encrypt DEK: <3ms
  - Logging: <1ms (async)

---

### 2. decryptDocument()

Decrypt document using envelope encryption.

#### Signature

```typescript
decryptDocument(request: DecryptRequest): Promise<DecryptedDocument>
```

#### Behavior

1. **Decrypt DEK**:
   - Decrypt DEK with Master Key via KMS

2. **Decrypt Document**:
   - Decrypt with AES-256-GCM using DEK
   - Verify authentication tag

3. **Log**:
   - Insert into `encryption_audit_log`

4. **Return**:
   - Return plaintext document

#### Algorithm

```typescript
async decryptDocument(request: DecryptRequest): Promise<DecryptedDocument> {
  // 1. Decrypt DEK with Master Key
  const dek = await this.decryptDEK(request.encrypted_dek);

  // 2. Decrypt document with DEK
  const decipher = createDecipheriv('aes-256-gcm', dek, request.iv);
  decipher.setAuthTag(request.auth_tag);

  let plaintext = decipher.update(request.ciphertext);
  plaintext = Buffer.concat([plaintext, decipher.final()]);

  // 3. Log
  await this.logEncryption({
    operation: 'decrypt',
    document_id: request.document_id,
    key_version: request.encrypted_dek.key_id,
    success: true
  });

  // 4. Return
  return {
    plaintext: plaintext.toString('utf8'),
    document_id: request.document_id,
    decrypted_at: new Date().toISOString(),
    key_version: request.encrypted_dek.key_id
  };
}
```

#### Example

```typescript
// Retrieve from database
const row = await db.query(`
  SELECT encrypted_dek, iv, auth_tag, kms_key_id
  FROM encrypted_documents
  WHERE document_id = $1
`, ['patient_record_123']);

// Retrieve ciphertext from blob storage
const ciphertext = await s3.getObject({
  Bucket: 'encrypted-documents',
  Key: 'patient_record_123'
}).Body;

// Decrypt
const decrypted = await encryptionEngine.decryptDocument({
  ciphertext,
  encrypted_dek: {
    ciphertext_blob: row.encrypted_dek,
    key_id: row.kms_key_id
  },
  iv: row.iv,
  auth_tag: row.auth_tag,
  document_id: 'patient_record_123'
});

console.log(decrypted.plaintext); // "Sensitive medical record..."
```

#### Performance

- **Target Latency**: <10ms p95 for 5MB document
- **Breakdown**:
  - KMS decrypt DEK: <3ms
  - AES-256-GCM decryption: <5ms (5MB)
  - Logging: <1ms (async)

---

### 3. rotateMasterKey()

Rotate Master Key with zero downtime.

#### Signature

```typescript
rotateMasterKey(options?: RotationOptions): Promise<RotationResult>
```

#### Behavior

1. **Create New Master Key**:
   - Create new KMS key (or use provided key)

2. **Update Configuration**:
   - Update application config to use new key for encryption
   - Old key still used for decryption (dual-key period)

3. **Background Re-Encryption** (Optional):
   - Re-encrypt all documents with new key
   - Can be done lazily or in background

4. **Retire Old Key**:
   - After all documents re-encrypted, retire old key

5. **Log**:
   - Insert into `key_rotation_history`

6. **Return**:
   - Return rotation result

#### Algorithm

```typescript
async rotateMasterKey(options?: RotationOptions): Promise<RotationResult> {
  const oldKeyId = this.currentMasterKeyId;

  // 1. Create new Master Key
  let newKeyId: string;

  if (options?.new_key_id) {
    newKeyId = options.new_key_id;
  } else {
    newKeyId = await this.kmsClient.createKey({
      Description: `Master Key rotated from ${oldKeyId}`,
      KeyUsage: 'ENCRYPT_DECRYPT'
    });
  }

  // 2. Update configuration
  this.currentMasterKeyId = newKeyId;

  // 3. Background re-encryption (lazy)
  let documentsReencrypted = 0;

  if (options?.force) {
    // Re-encrypt all documents immediately
    const documents = await this.getAllEncryptedDocuments();

    for (const doc of documents) {
      await this.reencryptDocument(doc.document_id, newKeyId);
      documentsReencrypted++;
    }
  }

  // 4. Log rotation
  await this.pool.query(`
    INSERT INTO key_rotation_history (old_key_id, new_key_id, documents_reencrypted)
    VALUES ($1, $2, $3)
  `, [oldKeyId, newKeyId, documentsReencrypted]);

  return {
    old_key_id: oldKeyId,
    new_key_id: newKeyId,
    rotated_at: new Date().toISOString(),
    documents_reencrypted: documentsReencrypted
  };
}

async reencryptDocument(documentId: string, newKeyId: string): Promise<void> {
  // 1. Decrypt with old key
  const decrypted = await this.decryptDocument({
    // ... retrieve from database
  });

  // 2. Encrypt with new key
  const encrypted = await this.encryptDocument({
    document: decrypted.plaintext,
    document_id: documentId
  });

  // 3. Update database
  await this.pool.query(`
    UPDATE encrypted_documents
    SET encrypted_dek = $1, kms_key_id = $2, iv = $3, auth_tag = $4
    WHERE document_id = $5
  `, [
    encrypted.encrypted_dek.ciphertext_blob,
    newKeyId,
    encrypted.iv,
    encrypted.auth_tag,
    documentId
  ]);
}
```

#### Example

```typescript
// Rotate Master Key (90 days)
const result = await encryptionEngine.rotateMasterKey();

console.log(`Rotated from ${result.old_key_id} to ${result.new_key_id}`);
console.log(`Re-encrypted ${result.documents_reencrypted} documents`);

// Scheduled rotation (cron job)
cron.schedule('0 0 * * 0', async () => { // Every Sunday
  const stats = await encryptionEngine.getStats();

  if (stats.days_since_rotation >= 90) {
    console.log('Rotating Master Key...');
    await encryptionEngine.rotateMasterKey({ force: true });
  }
});
```

#### Performance

- **Rotation Overhead**: <1s (configuration update)
- **Re-Encryption**: Lazy (background) or forced (minutes for large datasets)

---

## Envelope Encryption

### Why Envelope Encryption?

**Problem with Direct KMS Encryption**:
- Every encrypt/decrypt operation calls KMS (network latency)
- KMS has rate limits (1000-10000 ops/sec)
- Expensive ($0.03 per 10K requests)

**Solution: Envelope Encryption**:
1. Generate local DEK (fast, no network call)
2. Encrypt data locally with DEK (fast, AES-256-GCM)
3. Encrypt DEK with Master Key (1 KMS call per document/batch)

**Benefits**:
- **Performance**: <10ms vs 50-100ms per document
- **Cost**: 1 KMS call vs N KMS calls
- **Scalability**: No KMS rate limit issues

### Envelope Encryption Diagram

```
Plaintext Document (5MB)
         |
         | AES-256-GCM encrypt with DEK
         v
Ciphertext (5MB) + Auth Tag (16 bytes)
         |
         | Store in S3/blob storage
         v
     [Storage]

DEK (256 bits)
    |
    | KMS encrypt with Master Key
    v
Encrypted DEK (512 bytes)
    |
    | Store in database
    v
[PostgreSQL]

Master Key (256 bits)
    |
    | Never leaves KMS
    v
  [AWS KMS / GCP Cloud KMS / Azure Key Vault]
```

---

## Key Management

### Master Key Storage

**AWS KMS**:
```typescript
import { KMSClient, EncryptCommand, DecryptCommand } from '@aws-sdk/client-kms';

const kmsClient = new KMSClient({ region: 'us-east-1' });

async function encryptDEK(dek: Buffer, keyId: string): Promise<EncryptedDEK> {
  const command = new EncryptCommand({
    KeyId: keyId,
    Plaintext: dek
  });

  const response = await kmsClient.send(command);

  return {
    ciphertext_blob: Buffer.from(response.CiphertextBlob!),
    key_id: keyId
  };
}

async function decryptDEK(encrypted_dek: EncryptedDEK): Promise<Buffer> {
  const command = new DecryptCommand({
    CiphertextBlob: encrypted_dek.ciphertext_blob
  });

  const response = await kmsClient.send(command);

  return Buffer.from(response.Plaintext!);
}
```

**GCP Cloud KMS**:
```typescript
import { KeyManagementServiceClient } from '@google-cloud/kms';

const kmsClient = new KeyManagementServiceClient();

async function encryptDEK(dek: Buffer, keyName: string): Promise<EncryptedDEK> {
  const [response] = await kmsClient.encrypt({
    name: keyName,
    plaintext: dek
  });

  return {
    ciphertext_blob: Buffer.from(response.ciphertext!),
    key_id: keyName
  };
}

async function decryptDEK(encrypted_dek: EncryptedDEK): Promise<Buffer> {
  const [response] = await kmsClient.decrypt({
    name: encrypted_dek.key_id,
    ciphertext: encrypted_dek.ciphertext_blob
  });

  return Buffer.from(response.plaintext!);
}
```

**Azure Key Vault**:
```typescript
import { CryptographyClient } from '@azure/keyvault-keys';

const cryptoClient = new CryptographyClient(keyId, credential);

async function encryptDEK(dek: Buffer, keyId: string): Promise<EncryptedDEK> {
  const result = await cryptoClient.encrypt('RSA-OAEP-256', dek);

  return {
    ciphertext_blob: result.result,
    key_id: keyId
  };
}

async function decryptDEK(encrypted_dek: EncryptedDEK): Promise<Buffer> {
  const result = await cryptoClient.decrypt('RSA-OAEP-256', encrypted_dek.ciphertext_blob);

  return Buffer.from(result.result);
}
```

---

## Key Rotation

### Rotation Strategy

**90-Day Rotation**:
- Rotate Master Key every 90 days
- Lazy re-encryption (re-encrypt on access)
- Or forced re-encryption (background job)

**Graceful Migration**:
1. Create new Master Key
2. Update config to use new key for encryption
3. Old key still used for decryption (dual-key period)
4. Re-encrypt documents with new key (lazy or forced)
5. Retire old key after all documents migrated

### Lazy Re-Encryption

```typescript
async decryptDocument(request: DecryptRequest): Promise<DecryptedDocument> {
  // Decrypt with old key
  const decrypted = await this.decryptDocumentInternal(request);

  // Check if key is old (>90 days)
  const keyAge = this.getKeyAge(request.encrypted_dek.key_id);

  if (keyAge > 90) {
    // Re-encrypt with new key (lazy)
    await this.reencryptDocument(request.document_id, this.currentMasterKeyId);
  }

  return decrypted;
}
```

### Forced Re-Encryption

```typescript
async rotateMasterKey(options: RotationOptions): Promise<RotationResult> {
  // ... rotation logic

  if (options.force) {
    // Background job: re-encrypt all documents
    const documents = await this.getAllEncryptedDocuments();

    for (const doc of documents) {
      await this.reencryptDocument(doc.document_id, newKeyId);
    }
  }
}
```

---

## Edge Cases

### Edge Case 1: KMS Unavailable

**Scenario**: KMS is down or unreachable.

**Resolution**:
- Retry with exponential backoff (3 attempts)
- Fall back to cached DEK if available (read-only mode)
- Alert operations team

```typescript
async encryptDEK(dek: Buffer): Promise<EncryptedDEK> {
  let attempt = 0;
  const maxAttempts = 3;

  while (attempt < maxAttempts) {
    try {
      return await this.kmsClient.encrypt(dek);
    } catch (error) {
      attempt++;
      if (attempt >= maxAttempts) throw error;

      await sleep(Math.pow(2, attempt) * 1000); // Exponential backoff
    }
  }
}
```

---

### Edge Case 2: Corrupted Ciphertext

**Scenario**: Ciphertext or auth tag is corrupted.

**Resolution**:
```typescript
async decryptDocument(request: DecryptRequest): Promise<DecryptedDocument> {
  try {
    // Decrypt
    const decipher = createDecipheriv('aes-256-gcm', dek, request.iv);
    decipher.setAuthTag(request.auth_tag);

    let plaintext = decipher.update(request.ciphertext);
    plaintext = Buffer.concat([plaintext, decipher.final()]);

    return { plaintext: plaintext.toString('utf8'), ... };
  } catch (error) {
    if (error.message.includes('authentication')) {
      throw new CorruptedDataError('Ciphertext authentication failed');
    }
    throw error;
  }
}
```

---

### Edge Case 3: Wrong DEK Used for Decryption

**Scenario**: Encrypted with DEK A, trying to decrypt with DEK B.

**Resolution**: Authentication tag verification will fail. Same as corrupted ciphertext.

---

### Edge Case 4: Key Rotation During Decryption

**Scenario**: Key rotated while document being decrypted.

**Resolution**: Use key version stored with document. Old key still available for decryption.

---

### Edge Case 5: Very Large Documents (>100MB)

**Scenario**: Document exceeds reasonable size for in-memory encryption.

**Resolution**:
- Stream encryption/decryption
- Process in chunks

```typescript
async encryptLargeDocument(stream: Readable): Promise<EncryptedDocument> {
  const dek = await this.generateDEK();
  const iv = randomBytes(12);
  const cipher = createCipheriv('aes-256-gcm', dek, iv);

  const outputStream = new PassThrough();

  stream.pipe(cipher).pipe(outputStream);

  return {
    ciphertext: outputStream, // Stream
    encrypted_dek: await this.encryptDEK(dek),
    iv,
    auth_tag: cipher.getAuthTag(), // Get after stream completes
    ...
  };
}
```

---

### Edge Case 6: Multiple Concurrent Rotations

**Scenario**: Two rotation jobs triggered simultaneously.

**Resolution**: Use database lock to ensure only one rotation at a time.

```typescript
async rotateMasterKey(): Promise<RotationResult> {
  // Acquire lock
  await this.pool.query(`SELECT pg_advisory_lock(12345)`);

  try {
    // Rotation logic
    // ...
  } finally {
    // Release lock
    await this.pool.query(`SELECT pg_advisory_unlock(12345)`);
  }
}
```

---

### Edge Case 7: Missing Encryption Metadata

**Scenario**: Document exists but encryption metadata (iv, auth_tag) missing.

**Resolution**: Document cannot be decrypted. Log error, alert, mark document as corrupted.

---

## Performance Characteristics

### Latency Targets

| Operation | Document Size | Target Latency (p95) | Notes |
|-----------|--------------|---------------------|-------|
| `encryptDocument()` | 1KB | < 3ms | Small document |
| `encryptDocument()` | 1MB | < 8ms | Medium document |
| `encryptDocument()` | 5MB | < 10ms | Large document |
| `decryptDocument()` | 1KB | < 3ms | Small document |
| `decryptDocument()` | 1MB | < 8ms | Medium document |
| `decryptDocument()` | 5MB | < 10ms | Large document |
| `rotateMasterKey()` | - | < 1s | Configuration update only |

### Throughput

- **Encryption/sec**: 1K documents/sec (1MB each)
- **Decryption/sec**: 1K documents/sec (1MB each)
- **KMS calls/sec**: <100 calls/sec (batched DEK encryption)

### Latency Breakdown

**Encryption (5MB document)**:
- DEK generation: 0.5ms
- AES-256-GCM encryption: 5ms
- KMS encrypt DEK: 3ms
- Logging (async): 1ms
- **Total**: ~10ms p95

**Decryption (5MB document)**:
- KMS decrypt DEK: 3ms
- AES-256-GCM decryption: 5ms
- Logging (async): 1ms
- **Total**: ~10ms p95

---

## Implementation Notes

### Node.js Crypto Implementation

```typescript
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';
import { KMSClient, EncryptCommand, DecryptCommand } from '@aws-sdk/client-kms';

export class AWSKMSEncryptionEngine implements EncryptionEngine {
  private kmsClient: KMSClient;
  private masterKeyId: string;

  constructor(masterKeyId: string, region: string) {
    this.masterKeyId = masterKeyId;
    this.kmsClient = new KMSClient({ region });
  }

  async encryptDocument(request: EncryptRequest): Promise<EncryptedDocument> {
    // Implementation shown in Core Functionality section
    // ...
  }

  async decryptDocument(request: DecryptRequest): Promise<DecryptedDocument> {
    // Implementation shown in Core Functionality section
    // ...
  }

  async generateDEK(): Promise<Buffer> {
    return randomBytes(32); // 256 bits
  }

  async encryptDEK(dek: Buffer): Promise<EncryptedDEK> {
    const command = new EncryptCommand({
      KeyId: this.masterKeyId,
      Plaintext: dek
    });

    const response = await this.kmsClient.send(command);

    return {
      ciphertext_blob: Buffer.from(response.CiphertextBlob!),
      key_id: this.masterKeyId
    };
  }

  async decryptDEK(encrypted_dek: EncryptedDEK): Promise<Buffer> {
    const command = new DecryptCommand({
      CiphertextBlob: encrypted_dek.ciphertext_blob
    });

    const response = await this.kmsClient.send(command);

    return Buffer.from(response.Plaintext!);
  }
}
```

---

## Security Considerations

### 1. Key Storage

**Never Store Master Key in Application**:
- Master Key stored only in KMS
- Application never has access to Master Key plaintext

### 2. DEK Lifecycle

**DEK Security**:
- DEK generated randomly per document/batch
- DEK never stored in plaintext
- DEK encrypted with Master Key before storage

### 3. Authentication

**Use Authenticated Encryption (GCM)**:
- AES-256-GCM provides confidentiality + integrity
- Authentication tag prevents tampering

### 4. IV Generation

**Use Random IV**:
```typescript
const iv = randomBytes(12); // 12 bytes for GCM (never reuse!)
```

---

## Integration Patterns

### Pattern 1: Transparent Encryption in Storage Layer

```typescript
class EncryptedDocumentStore {
  constructor(
    private encryptionEngine: EncryptionEngine,
    private blobStorage: BlobStorage,
    private db: Database
  ) {}

  async save(documentId: string, content: string): Promise<void> {
    // Encrypt
    const encrypted = await this.encryptionEngine.encryptDocument({
      document: content,
      document_id: documentId
    });

    // Store ciphertext in blob storage
    await this.blobStorage.put(documentId, encrypted.ciphertext);

    // Store metadata in database
    await this.db.query(`
      INSERT INTO encrypted_documents (document_id, encrypted_dek, iv, auth_tag, kms_key_id)
      VALUES ($1, $2, $3, $4, $5)
    `, [
      documentId,
      encrypted.encrypted_dek.ciphertext_blob,
      encrypted.iv,
      encrypted.auth_tag,
      encrypted.key_version
    ]);
  }

  async get(documentId: string): Promise<string> {
    // Retrieve metadata
    const metadata = await this.db.query(`
      SELECT encrypted_dek, iv, auth_tag, kms_key_id
      FROM encrypted_documents
      WHERE document_id = $1
    `, [documentId]);

    // Retrieve ciphertext
    const ciphertext = await this.blobStorage.get(documentId);

    // Decrypt
    const decrypted = await this.encryptionEngine.decryptDocument({
      ciphertext,
      encrypted_dek: {
        ciphertext_blob: metadata.encrypted_dek,
        key_id: metadata.kms_key_id
      },
      iv: metadata.iv,
      auth_tag: metadata.auth_tag,
      document_id: documentId
    });

    return decrypted.plaintext;
  }
}
```

---

## Multi-Domain Examples

(Already covered in "Multi-Domain Applicability" section with 7 domains)

---

## Testing Strategy

### Unit Tests

```typescript
describe('EncryptionEngine', () => {
  let engine: EncryptionEngine;

  beforeEach(() => {
    engine = new AWSKMSEncryptionEngine(TEST_KMS_KEY_ID, 'us-east-1');
  });

  describe('encryptDocument()', () => {
    it('should encrypt and decrypt document', async () => {
      const plaintext = 'Sensitive data';

      // Encrypt
      const encrypted = await engine.encryptDocument({
        document: plaintext,
        document_id: 'doc_123'
      });

      expect(encrypted.ciphertext).toBeDefined();
      expect(encrypted.encrypted_dek).toBeDefined();

      // Decrypt
      const decrypted = await engine.decryptDocument({
        ciphertext: encrypted.ciphertext,
        encrypted_dek: encrypted.encrypted_dek,
        iv: encrypted.iv,
        auth_tag: encrypted.auth_tag,
        document_id: 'doc_123'
      });

      expect(decrypted.plaintext).toBe(plaintext);
    });

    it('should fail decryption with wrong auth tag', async () => {
      const plaintext = 'Sensitive data';

      const encrypted = await engine.encryptDocument({
        document: plaintext,
        document_id: 'doc_123'
      });

      // Corrupt auth tag
      encrypted.auth_tag[0] = encrypted.auth_tag[0] ^ 0xFF;

      await expect(
        engine.decryptDocument({
          ciphertext: encrypted.ciphertext,
          encrypted_dek: encrypted.encrypted_dek,
          iv: encrypted.iv,
          auth_tag: encrypted.auth_tag,
          document_id: 'doc_123'
        })
      ).rejects.toThrow('authentication');
    });
  });

  describe('rotateMasterKey()', () => {
    it('should rotate master key', async () => {
      const result = await engine.rotateMasterKey();

      expect(result.old_key_id).toBeDefined();
      expect(result.new_key_id).toBeDefined();
      expect(result.old_key_id).not.toBe(result.new_key_id);
    });
  });
});
```

### Performance Tests

```typescript
describe('EncryptionEngine Performance', () => {
  it('should encrypt 5MB document in <10ms', async () => {
    const largeDocument = Buffer.alloc(5 * 1024 * 1024, 'a'); // 5MB

    const startTime = Date.now();

    await engine.encryptDocument({
      document: largeDocument,
      document_id: 'large_doc'
    });

    const duration = Date.now() - startTime;

    console.log(`Encryption duration: ${duration}ms`);
    expect(duration).toBeLessThan(10);
  });

  it('should decrypt 5MB document in <10ms', async () => {
    const largeDocument = Buffer.alloc(5 * 1024 * 1024, 'a');

    const encrypted = await engine.encryptDocument({
      document: largeDocument,
      document_id: 'large_doc'
    });

    const startTime = Date.now();

    await engine.decryptDocument({
      ciphertext: encrypted.ciphertext,
      encrypted_dek: encrypted.encrypted_dek,
      iv: encrypted.iv,
      auth_tag: encrypted.auth_tag,
      document_id: 'large_doc'
    });

    const duration = Date.now() - startTime;

    console.log(`Decryption duration: ${duration}ms`);
    expect(duration).toBeLessThan(10);
  });
});
```

---

## Configuration

### Tunable Parameters

```typescript
interface EncryptionEngineConfig {
  // KMS
  kms_provider: 'aws' | 'gcp' | 'azure';
  master_key_id: string;
  kms_region?: string;              // AWS/GCP only

  // Key Rotation
  rotation_interval_days: number;   // Default: 90
  auto_rotate: boolean;             // Default: true
  lazy_reencryption: boolean;       // Default: true

  // Performance
  dek_cache_enabled: boolean;       // Default: false (security risk)
  dek_cache_ttl_seconds: number;    // Default: 60

  // Audit
  log_all_operations: boolean;      // Default: false (too verbose)
  log_failures: boolean;            // Default: true
}
```

---

## Observability

### Metrics

```typescript
interface EncryptionMetrics {
  documents_encrypted_total: Counter;
  documents_decrypted_total: Counter;
  encryption_latency_ms: Histogram;
  decryption_latency_ms: Histogram;
  kms_calls_total: Counter;
  kms_errors_total: Counter;
  key_rotation_events_total: Counter;
}
```

---

## Future Enhancements

### Phase 1: Client-Side Encryption (6 months)
- Encrypt in browser before upload
- Zero-knowledge encryption

### Phase 2: Hardware Security Modules (HSM) (9 months)
- Support on-premise HSM
- FIPS 140-2 Level 3 compliance

### Phase 3: Quantum-Resistant Encryption (12+ months)
- Post-quantum cryptography algorithms
- Future-proof against quantum computers

---

**End of EncryptionEngine OL Primitive Specification**
