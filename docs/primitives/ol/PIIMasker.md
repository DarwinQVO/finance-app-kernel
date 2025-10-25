# PIIMasker OL Primitive

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
8. [Detection Strategies](#detection-strategies)
9. [Masking Strategies](#masking-strategies)
10. [Pattern Library](#pattern-library)
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

The **PIIMasker** is a high-performance primitive that detects and masks Personally Identifiable Information (PII) in observations before they are stored, displayed, or exported. It uses pattern-based detection (regex) and semantic rules (field name analysis) to identify sensitive data, then applies configurable masking strategies to protect privacy while maintaining data utility.

### Key Capabilities

- **Pattern-Based Detection**: Regex patterns for SSN, credit card, email, phone, IP addresses
- **Semantic Detection**: Field name analysis (e.g., "ssn", "credit_card_number")
- **Multiple Masking Strategies**: Partial masking, full masking, tokenization
- **High Performance**: <5ms p95 per observation masking
- **Reversible Tokenization**: Secure token generation for data linking
- **Customizable Patterns**: Add domain-specific PII patterns
- **Audit Trail**: Logs all masking operations for compliance

### Design Philosophy

The PIIMasker follows four core principles:

1. **Privacy First**: Mask PII by default, unmask only when authorized
2. **Performance**: <5ms p95 latency per observation
3. **Configurability**: Support domain-specific PII patterns
4. **Auditability**: Log all masking/unmasking operations

---

## Purpose & Scope

### Problem Statement

In document processing systems, PII poses critical risks:

- **Regulatory Compliance**: GDPR, HIPAA, CCPA require PII protection
- **Data Breaches**: Exposed PII leads to financial and reputational damage
- **Over-Collection**: Systems collect more PII than necessary
- **Inadequate Masking**: Manual redaction is error-prone and slow
- **Display Exposure**: PII visible in UIs, logs, and exports

Traditional approaches have significant limitations:

**Approach 1: Manual Redaction**
- ❌ Slow (minutes per document)
- ❌ Error-prone (missed PII)
- ❌ Not scalable

**Approach 2: Blacklist Fields**
- ❌ Brittle (field names change)
- ❌ Misses unstructured PII
- ❌ All-or-nothing (no partial masking)

**Approach 3: PII Masker Primitive**
- ✅ Automatic detection via regex + semantic rules
- ✅ <5ms p95 per observation
- ✅ Multiple masking strategies
- ✅ Reversible tokenization
- ✅ Full audit trail

### Solution

The PIIMasker implements a **multi-strategy PII protection system** with:

1. **Pattern Library**: 20+ built-in patterns (SSN, credit card, email, phone, etc.)
2. **Semantic Rules**: Field name detection (e.g., "ssn" → mask as SSN)
3. **Masking Strategies**: Partial (show last 4), full (***), tokenization
4. **Token Store**: Secure token↔value mapping for reversibility
5. **Audit Log**: All masking operations logged

---

## Multi-Domain Applicability

The PIIMasker is universally applicable across domains. Below are 7+ domain-specific use cases:

### 1. Finance

**Use Case**: Mask SSNs, account numbers, credit card numbers in financial documents.

**PII Types**:
- Social Security Numbers (SSN): 123-45-6789
- Account Numbers: 12345678901234
- Credit Card Numbers: 4111-1111-1111-1111
- Routing Numbers: 021000021
- Tax IDs (EIN): 12-3456789

**Masking Strategy**: Partial masking (show last 4 digits)

**Example**:
```typescript
// Mask bank statement observation
const observation = {
  field: "account_number",
  value: "123456789012",
  upload_id: "upload_123"
};

const masked = await masker.maskObservation(observation, {
  strategy: "partial",
  show_last_n: 4
});

console.log(masked.value); // "********9012"
console.log(masked.pii_detected); // true
console.log(masked.pii_type); // "account_number"
console.log(masked.token); // "tok_abc123def456" (for reversible unmasking)

// Unmask with proper authorization
const unmasked = await masker.unmaskObservation(masked.token, {
  user_id: "user_123",
  reason: "customer_service_inquiry"
});

console.log(unmasked.value); // "123456789012"
```

**Performance Target**: <5ms p95 per observation.

---

### 2. Healthcare

**Use Case**: Mask patient identifiers, medical record numbers, health insurance IDs.

**PII Types**:
- Medical Record Numbers (MRN): MRN-123456
- Health Insurance IDs: H12345678
- Patient Names: John Doe
- Date of Birth: 1990-01-15
- Phone Numbers: (555) 123-4567

**Masking Strategy**: Full masking for patient names, partial for MRN

**Example**:
```typescript
// Mask HL7 message observation
const observation = {
  field: "patient_name",
  value: "John Michael Doe",
  upload_id: "msg_456"
};

const masked = await masker.maskObservation(observation, {
  strategy: "full"
});

console.log(masked.value); // "[REDACTED]"

// Mask MRN with partial strategy
const mrnObs = {
  field: "medical_record_number",
  value: "MRN-123456",
  upload_id: "msg_456"
};

const maskedMRN = await masker.maskObservation(mrnObs, {
  strategy: "partial",
  show_last_n: 4
});

console.log(maskedMRN.value); // "MRN-***456"
```

**Performance Target**: <5ms p95 per observation, HIPAA compliant.

---

### 3. Legal

**Use Case**: Mask party names, SSNs, attorney-client privileged information.

**PII Types**:
- Party Names: Plaintiff John Doe
- Attorney Names: Jane Smith, Esq.
- Case Numbers: 2023-CV-12345
- SSNs in court documents
- Addresses: 123 Main St, Anytown, USA

**Masking Strategy**: Full masking for names, tokenization for case linking

**Example**:
```typescript
// Mask contract observation
const observation = {
  field: "party_name",
  value: "John Michael Doe",
  upload_id: "doc_789"
};

const masked = await masker.maskObservation(observation, {
  strategy: "tokenize"
});

console.log(masked.value); // "tok_party_abc123"
console.log(masked.token); // "tok_party_abc123"

// Same party in another document gets same token
const observation2 = {
  field: "party_name",
  value: "John Michael Doe",
  upload_id: "doc_790"
};

const masked2 = await masker.maskObservation(observation2, {
  strategy: "tokenize"
});

console.log(masked2.token === masked.token); // true (consistent tokenization)
```

**Performance Target**: <5ms p95 per observation, support multi-document linking.

---

### 4. HR (Human Resources)

**Use Case**: Mask employee SSNs, salaries, performance reviews.

**PII Types**:
- Employee SSNs: 123-45-6789
- Salaries: $125,000.00
- Performance ratings: 3.5/5.0
- Home addresses
- Emergency contact phone numbers

**Masking Strategy**: Full masking for salaries, partial for SSN

**Example**:
```typescript
// Mask payroll observation
const observation = {
  field: "employee_ssn",
  value: "123-45-6789",
  upload_id: "payroll_123"
};

const masked = await masker.maskObservation(observation, {
  strategy: "partial",
  show_last_n: 4
});

console.log(masked.value); // "***-**-6789"

// Mask salary with full strategy
const salaryObs = {
  field: "annual_salary",
  value: "125000.00",
  upload_id: "payroll_123"
};

const maskedSalary = await masker.maskObservation(salaryObs, {
  strategy: "full"
});

console.log(maskedSalary.value); // "[REDACTED]"
```

**Performance Target**: <5ms p95, GDPR compliant.

---

### 5. E-commerce

**Use Case**: Mask customer credit cards, shipping addresses, email addresses.

**PII Types**:
- Credit Card Numbers: 4111-1111-1111-1111
- CVV Codes: 123
- Email Addresses: customer@example.com
- Shipping Addresses: 123 Main St
- Phone Numbers: (555) 123-4567

**Masking Strategy**: Partial for credit cards (show last 4), full for CVV

**Example**:
```typescript
// Mask order observation
const observation = {
  field: "credit_card_number",
  value: "4111111111111111",
  upload_id: "order_567"
};

const masked = await masker.maskObservation(observation, {
  strategy: "partial",
  show_last_n: 4
});

console.log(masked.value); // "************1111"

// Mask email address
const emailObs = {
  field: "customer_email",
  value: "john.doe@example.com",
  upload_id: "order_567"
};

const maskedEmail = await masker.maskObservation(emailObs, {
  strategy: "partial",
  show_last_n: 10 // Show domain
});

console.log(maskedEmail.value); // "j***@example.com"
```

**Performance Target**: <5ms p95, PCI DSS compliant.

---

### 6. SaaS

**Use Case**: Mask API keys, webhook secrets, authentication tokens.

**PII Types**:
- API Keys: tc_live_abc123def456
- Webhook Secrets: whsec_abc123
- OAuth Tokens: oauth_token_abc123
- IP Addresses: 192.168.1.1
- Session IDs: sess_abc123def456

**Masking Strategy**: Full masking for secrets, partial for IP addresses

**Example**:
```typescript
// Mask webhook observation
const observation = {
  field: "api_key",
  value: "tc_live_abc123def456ghi789",
  upload_id: "webhook_890"
};

const masked = await masker.maskObservation(observation, {
  strategy: "full"
});

console.log(masked.value); // "[REDACTED]"

// Mask IP address with partial strategy
const ipObs = {
  field: "client_ip",
  value: "192.168.1.100",
  upload_id: "webhook_890"
};

const maskedIP = await masker.maskObservation(ipObs, {
  strategy: "partial",
  show_last_n: 3 // Show last octet
});

console.log(maskedIP.value); // "192.168.***.100"
```

**Performance Target**: <3ms p95 per observation.

---

### 7. Government

**Use Case**: Mask taxpayer IDs, passport numbers, driver's license numbers.

**PII Types**:
- Taxpayer IDs: 123-45-6789
- Passport Numbers: N12345678
- Driver's License Numbers: DL-12345678
- Voter Registration Numbers
- Government Employee IDs

**Masking Strategy**: Full masking for government IDs

**Example**:
```typescript
// Mask tax return observation
const observation = {
  field: "taxpayer_id",
  value: "123-45-6789",
  upload_id: "tax_return_345"
};

const masked = await masker.maskObservation(observation, {
  strategy: "full"
});

console.log(masked.value); // "[REDACTED]"

// Mask passport number
const passportObs = {
  field: "passport_number",
  value: "N12345678",
  upload_id: "travel_doc_456"
};

const maskedPassport = await masker.maskObservation(passportObs, {
  strategy: "tokenize"
});

console.log(maskedPassport.value); // "tok_passport_abc123"
```

**Performance Target**: <5ms p95, FISMA compliant.

---

### Cross-Domain Benefits

**Consistent Privacy Protection**: All domains benefit from:
- Automatic PII detection and masking
- Configurable masking strategies per domain
- Audit trail for compliance
- Reversible tokenization for data linking

**Regulatory Compliance**: Supports:
- GDPR (Privacy): Right to erasure, data minimization
- HIPAA (Healthcare): PHI protection
- PCI DSS (Payments): Credit card masking
- CCPA (California Privacy): Consumer data protection

---

## Core Concepts

### PII Categories

The PIIMasker detects 10 primary PII categories:

1. **Identification Numbers**: SSN, passport, driver's license, tax ID
2. **Financial Data**: Credit card, account number, routing number
3. **Contact Information**: Email, phone, address
4. **Healthcare Data**: MRN, health insurance ID, patient name
5. **Authentication Secrets**: API key, password, token, secret
6. **Network Identifiers**: IP address, MAC address
7. **Personal Attributes**: Name, date of birth, age
8. **Biometric Data**: Fingerprint, retina scan (future)
9. **Location Data**: GPS coordinates, postal code
10. **Government IDs**: Taxpayer ID, voter registration

### Detection Methods

**1. Pattern-Based Detection** (Regex)
- Match value against regex patterns
- Example: SSN pattern `^\d{3}-\d{2}-\d{4}$`

**2. Semantic Detection** (Field Names)
- Analyze field name for PII indicators
- Example: `"ssn"` → SSN, `"email"` → Email

**3. Hybrid Detection**
- Combine pattern + semantic for higher accuracy
- Example: Field named `"ssn"` with value matching SSN pattern

### Masking Strategies

**1. Partial Masking**
- Show first/last N characters, mask middle
- Example: `"123-45-6789"` → `"***-**-6789"`

**2. Full Masking**
- Replace entire value with `"[REDACTED]"`
- Example: `"John Doe"` → `"[REDACTED]"`

**3. Tokenization**
- Replace value with secure token
- Example: `"123-45-6789"` → `"tok_ssn_abc123def456"`
- Token stored in secure token store for reversibility

---

## Interface Definition

### TypeScript Interface

```typescript
interface PIIMasker {
  /**
   * Mask PII in a single observation
   *
   * @param observation - Observation with potential PII
   * @param options - Masking options
   * @returns Masked observation with metadata
   * @throws ValidationError if observation is invalid
   */
  maskObservation(
    observation: Observation,
    options?: MaskingOptions
  ): Promise<MaskedObservation>;

  /**
   * Mask PII in multiple observations (batch)
   *
   * @param observations - Array of observations
   * @param options - Masking options
   * @returns Array of masked observations
   * @throws ValidationError if any observation is invalid
   */
  maskBatch(
    observations: Observation[],
    options?: MaskingOptions
  ): Promise<MaskedObservation[]>;

  /**
   * Detect PII in observation without masking
   *
   * @param observation - Observation to analyze
   * @returns PII detection result
   */
  detectPII(observation: Observation): Promise<PIIDetectionResult>;

  /**
   * Unmask observation using token
   *
   * @param token - Masking token
   * @param authorization - Authorization context
   * @returns Unmasked observation
   * @throws UnauthorizedError if not authorized
   */
  unmaskObservation(
    token: string,
    authorization: UnmaskAuthorization
  ): Promise<UnmaskedObservation>;

  /**
   * Register custom PII pattern
   *
   * @param pattern - Custom PII pattern definition
   * @returns Pattern ID
   */
  registerPattern(pattern: PIIPattern): Promise<string>;

  /**
   * Update masking strategy for PII type
   *
   * @param pii_type - PII type (e.g., "ssn")
   * @param strategy - Masking strategy
   */
  updateStrategy(pii_type: string, strategy: MaskingStrategy): Promise<void>;

  /**
   * Get masking statistics
   *
   * @returns Masking stats
   */
  getStats(): MaskingStats;
}
```

---

## Data Model

### Observation Type

```typescript
interface Observation {
  field: string;                  // Field name (e.g., "ssn", "email")
  value: string;                  // Field value
  upload_id: string;              // Upload identifier
  metadata?: Record<string, any>; // Additional context
}
```

### MaskedObservation Type

```typescript
interface MaskedObservation {
  // Original Fields
  field: string;
  upload_id: string;

  // Masked Data
  value: string;                  // Masked value
  masked: boolean;                // Whether masking was applied

  // PII Detection
  pii_detected: boolean;          // PII detected
  pii_type?: string;              // PII type (e.g., "ssn", "email")
  detection_method?: string;      // "pattern" | "semantic" | "hybrid"
  confidence_score?: number;      // Detection confidence (0.0 to 1.0)

  // Masking Metadata
  strategy?: MaskingStrategy;     // Strategy used
  token?: string;                 // Token for unmasking (if tokenize strategy)

  // Audit
  masked_at: string;              // ISO timestamp
  masked_by?: string;             // User/service that triggered masking
}
```

### MaskingOptions Type

```typescript
interface MaskingOptions {
  // Strategy
  strategy?: MaskingStrategy;     // "partial" | "full" | "tokenize"

  // Partial Masking Options
  show_first_n?: number;          // Show first N characters
  show_last_n?: number;           // Show last N characters
  mask_char?: string;             // Masking character (default: "*")

  // Detection Options
  detection_threshold?: number;   // Min confidence score (0.0 to 1.0)
  skip_semantic?: boolean;        // Skip semantic detection
  skip_pattern?: boolean;         // Skip pattern detection

  // Audit
  masked_by?: string;             // User/service triggering masking
}
```

### PIIDetectionResult Type

```typescript
interface PIIDetectionResult {
  pii_detected: boolean;
  pii_type?: string;
  detection_method?: string;
  confidence_score?: number;
  matched_pattern?: string;       // Regex pattern matched
  matched_field_name?: boolean;   // Field name matched
}
```

### UnmaskAuthorization Type

```typescript
interface UnmaskAuthorization {
  user_id: string;                // User requesting unmask
  reason: string;                 // Reason for unmasking
  role?: string;                  // User role
  ip_address?: string;            // Request IP address
}
```

### UnmaskedObservation Type

```typescript
interface UnmaskedObservation {
  field: string;
  value: string;                  // Original unmasked value
  token: string;                  // Token used for unmasking
  unmasked_at: string;            // ISO timestamp
  unmasked_by: string;            // User ID
  reason: string;                 // Unmasking reason
}
```

### PIIPattern Type

```typescript
interface PIIPattern {
  pattern_id?: string;            // Auto-generated
  pii_type: string;               // PII type (e.g., "ssn")
  pattern: string;                // Regex pattern
  description: string;            // Human-readable description
  field_name_indicators?: string[]; // Field names that indicate this PII type
  default_strategy?: MaskingStrategy; // Default masking strategy
  priority?: number;              // Pattern priority (higher = checked first)
}
```

### MaskingStrategy Enum

```typescript
enum MaskingStrategy {
  PARTIAL = "partial",            // Partial masking (show first/last N)
  FULL = "full",                  // Full masking ([REDACTED])
  TOKENIZE = "tokenize"           // Replace with token
}
```

### MaskingStats Type

```typescript
interface MaskingStats {
  // Throughput
  observations_masked_total: number;
  observations_per_second: number;

  // Detection
  pii_detected_total: number;
  pii_detection_rate: number;     // Percentage of observations with PII

  // By Type
  pii_by_type: Record<string, number>; // Count by PII type

  // Performance
  masking_latency_p50_ms: number;
  masking_latency_p95_ms: number;
  masking_latency_p99_ms: number;

  // Errors
  masking_errors_total: number;
}
```

### Database Schema (PostgreSQL)

```sql
-- Token store for reversible masking
CREATE TABLE pii_tokens (
  -- Primary Key
  token VARCHAR(64) PRIMARY KEY,

  -- Original Value (encrypted)
  value_encrypted BYTEA NOT NULL,

  -- PII Metadata
  pii_type VARCHAR(64) NOT NULL,
  field VARCHAR(128) NOT NULL,

  -- Timestamps
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMP WITH TIME ZONE
);

-- Masking audit log
CREATE TABLE pii_masking_log (
  -- Primary Key
  log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Observation
  upload_id VARCHAR(128) NOT NULL,
  field VARCHAR(128) NOT NULL,

  -- Masking
  pii_type VARCHAR(64),
  strategy VARCHAR(32) NOT NULL,
  token VARCHAR(64),

  -- Audit
  masked_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  masked_by VARCHAR(128)
);

-- Unmasking audit log
CREATE TABLE pii_unmasking_log (
  -- Primary Key
  log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Token
  token VARCHAR(64) NOT NULL,

  -- Authorization
  user_id VARCHAR(128) NOT NULL,
  reason TEXT NOT NULL,
  role VARCHAR(64),
  ip_address VARCHAR(45),

  -- Timestamp
  unmasked_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_pii_tokens_pii_type ON pii_tokens(pii_type);
CREATE INDEX idx_pii_tokens_expires_at ON pii_tokens(expires_at);
CREATE INDEX idx_masking_log_upload_id ON pii_masking_log(upload_id);
CREATE INDEX idx_masking_log_pii_type ON pii_masking_log(pii_type);
CREATE INDEX idx_unmasking_log_token ON pii_unmasking_log(token);
CREATE INDEX idx_unmasking_log_user_id ON pii_unmasking_log(user_id);
```

---

## Core Functionality

### 1. maskObservation()

Mask PII in a single observation.

#### Signature

```typescript
maskObservation(
  observation: Observation,
  options?: MaskingOptions
): Promise<MaskedObservation>
```

#### Behavior

1. **Detect PII**:
   - Run pattern-based detection (regex)
   - Run semantic detection (field name)
   - Combine results for final determination

2. **Determine Strategy**:
   - Use `options.strategy` if provided
   - Otherwise, use default strategy for PII type
   - Default: `partial` for financial data, `full` for names

3. **Apply Masking**:
   - **Partial**: Show first/last N characters
   - **Full**: Replace with `[REDACTED]`
   - **Tokenize**: Generate token, store in token store

4. **Log**:
   - Insert into `pii_masking_log` table

5. **Return**:
   - Return masked observation with metadata

#### Algorithm

```typescript
async maskObservation(
  observation: Observation,
  options?: MaskingOptions
): Promise<MaskedObservation> {
  // 1. Detect PII
  const detection = await this.detectPII(observation);

  // If no PII detected, return observation unchanged
  if (!detection.pii_detected) {
    return {
      field: observation.field,
      value: observation.value,
      upload_id: observation.upload_id,
      masked: false,
      pii_detected: false,
      masked_at: new Date().toISOString()
    };
  }

  // 2. Determine strategy
  const strategy = options?.strategy || this.getDefaultStrategy(detection.pii_type);

  // 3. Apply masking
  let maskedValue: string;
  let token: string | undefined;

  switch (strategy) {
    case MaskingStrategy.PARTIAL:
      maskedValue = this.applyPartialMasking(
        observation.value,
        options?.show_first_n || 0,
        options?.show_last_n || 4,
        options?.mask_char || '*'
      );
      break;

    case MaskingStrategy.FULL:
      maskedValue = '[REDACTED]';
      break;

    case MaskingStrategy.TOKENIZE:
      token = await this.generateToken(observation.value, detection.pii_type);
      maskedValue = token;
      break;
  }

  // 4. Log
  await this.logMasking({
    upload_id: observation.upload_id,
    field: observation.field,
    pii_type: detection.pii_type,
    strategy,
    token,
    masked_by: options?.masked_by
  });

  // 5. Return
  return {
    field: observation.field,
    value: maskedValue,
    upload_id: observation.upload_id,
    masked: true,
    pii_detected: true,
    pii_type: detection.pii_type,
    detection_method: detection.detection_method,
    confidence_score: detection.confidence_score,
    strategy,
    token,
    masked_at: new Date().toISOString(),
    masked_by: options?.masked_by
  };
}
```

#### Example

```typescript
const observation = {
  field: "ssn",
  value: "123-45-6789",
  upload_id: "upload_123"
};

const masked = await masker.maskObservation(observation, {
  strategy: "partial",
  show_last_n: 4
});

console.log(masked.value); // "***-**-6789"
console.log(masked.pii_detected); // true
console.log(masked.pii_type); // "ssn"
```

#### Performance

- **Target Latency**: <5ms p95

---

### 2. maskBatch()

Mask PII in multiple observations (batch).

#### Signature

```typescript
maskBatch(
  observations: Observation[],
  options?: MaskingOptions
): Promise<MaskedObservation[]>
```

#### Behavior

1. **Validate**:
   - Validate batch size <= 1000

2. **Process**:
   - Process each observation in parallel
   - Use `Promise.all()` for concurrency

3. **Return**:
   - Return array of masked observations

#### Example

```typescript
const observations = [
  { field: "ssn", value: "123-45-6789", upload_id: "upload_123" },
  { field: "email", value: "john@example.com", upload_id: "upload_123" },
  { field: "phone", value: "(555) 123-4567", upload_id: "upload_123" }
];

const masked = await masker.maskBatch(observations, {
  strategy: "partial",
  show_last_n: 4
});

masked.forEach(m => {
  console.log(`${m.field}: ${m.value} (PII: ${m.pii_detected})`);
});
```

#### Performance

- **Target Latency**: <100ms for 1000 observations
- **Throughput**: 10K observations/sec

---

### 3. detectPII()

Detect PII in observation without masking.

#### Signature

```typescript
detectPII(observation: Observation): Promise<PIIDetectionResult>
```

#### Behavior

1. **Pattern Detection**:
   - Match value against all registered patterns
   - Calculate confidence score based on match quality

2. **Semantic Detection**:
   - Check if field name matches known PII indicators
   - Example: `"ssn"` → SSN, `"email"` → Email

3. **Hybrid Score**:
   - Combine pattern + semantic scores
   - Threshold: 0.7 (70% confidence required)

4. **Return**:
   - Return detection result

#### Algorithm

```typescript
async detectPII(observation: Observation): Promise<PIIDetectionResult> {
  let maxConfidence = 0;
  let detectedType: string | undefined;
  let detectionMethod: string | undefined;
  let matchedPattern: string | undefined;

  // 1. Pattern detection
  for (const pattern of this.patterns) {
    const regex = new RegExp(pattern.pattern);
    if (regex.test(observation.value)) {
      const confidence = this.calculatePatternConfidence(
        observation.value,
        pattern
      );

      if (confidence > maxConfidence) {
        maxConfidence = confidence;
        detectedType = pattern.pii_type;
        detectionMethod = 'pattern';
        matchedPattern = pattern.pattern;
      }
    }
  }

  // 2. Semantic detection
  const semanticResult = this.detectSemantic(observation.field);
  if (semanticResult.confidence > maxConfidence) {
    maxConfidence = semanticResult.confidence;
    detectedType = semanticResult.pii_type;
    detectionMethod = 'semantic';
  }

  // 3. Hybrid detection
  if (detectionMethod === 'pattern' && semanticResult.pii_type === detectedType) {
    detectionMethod = 'hybrid';
    maxConfidence = Math.min(maxConfidence + 0.1, 1.0); // Boost confidence
  }

  // Threshold
  const pii_detected = maxConfidence >= 0.7;

  return {
    pii_detected,
    pii_type: pii_detected ? detectedType : undefined,
    detection_method: pii_detected ? detectionMethod : undefined,
    confidence_score: maxConfidence,
    matched_pattern: matchedPattern
  };
}
```

#### Example

```typescript
const observation = {
  field: "ssn",
  value: "123-45-6789",
  upload_id: "upload_123"
};

const detection = await masker.detectPII(observation);

console.log(detection.pii_detected); // true
console.log(detection.pii_type); // "ssn"
console.log(detection.detection_method); // "hybrid"
console.log(detection.confidence_score); // 0.95
```

#### Performance

- **Target Latency**: <3ms p95

---

### 4. unmaskObservation()

Unmask observation using token (requires authorization).

#### Signature

```typescript
unmaskObservation(
  token: string,
  authorization: UnmaskAuthorization
): Promise<UnmaskedObservation>
```

#### Behavior

1. **Validate Authorization**:
   - Check if user has permission to unmask
   - Verify reason is provided

2. **Retrieve Token**:
   - Query `pii_tokens` table
   - Decrypt value

3. **Log**:
   - Insert into `pii_unmasking_log` table

4. **Return**:
   - Return unmasked observation

#### Algorithm

```typescript
async unmaskObservation(
  token: string,
  authorization: UnmaskAuthorization
): Promise<UnmaskedObservation> {
  // 1. Validate authorization
  if (!authorization.user_id || !authorization.reason) {
    throw new ValidationError('user_id and reason are required');
  }

  // Check permission (integrate with AccessControl primitive)
  const hasPermission = await this.accessControl.checkPermission({
    user_id: authorization.user_id,
    resource: 'pii',
    action: 'unmask'
  });

  if (!hasPermission) {
    throw new UnauthorizedError('User does not have permission to unmask PII');
  }

  // 2. Retrieve token
  const query = `
    SELECT field, value_encrypted, pii_type
    FROM pii_tokens
    WHERE token = $1
  `;

  const result = await this.pool.query(query, [token]);

  if (result.rows.length === 0) {
    throw new NotFoundError(`Token not found: ${token}`);
  }

  const row = result.rows[0];

  // Decrypt value
  const value = await this.decrypt(row.value_encrypted);

  // 3. Log
  await this.logUnmasking({
    token,
    user_id: authorization.user_id,
    reason: authorization.reason,
    role: authorization.role,
    ip_address: authorization.ip_address
  });

  // 4. Return
  return {
    field: row.field,
    value,
    token,
    unmasked_at: new Date().toISOString(),
    unmasked_by: authorization.user_id,
    reason: authorization.reason
  };
}
```

#### Example

```typescript
const unmasked = await masker.unmaskObservation('tok_ssn_abc123', {
  user_id: 'user_123',
  reason: 'Customer service inquiry',
  role: 'customer_service_agent',
  ip_address: '192.168.1.100'
});

console.log(unmasked.value); // "123-45-6789"
console.log(unmasked.unmasked_by); // "user_123"
```

#### Performance

- **Target Latency**: <10ms p95

---

## Detection Strategies

### Pattern-Based Detection

Use regex patterns to match PII in values.

#### Built-In Patterns

```typescript
const BUILT_IN_PATTERNS: PIIPattern[] = [
  // SSN
  {
    pii_type: 'ssn',
    pattern: '^\\d{3}-\\d{2}-\\d{4}$',
    description: 'US Social Security Number (123-45-6789)',
    field_name_indicators: ['ssn', 'social_security', 'social_security_number'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 10
  },

  // Credit Card
  {
    pii_type: 'credit_card',
    pattern: '^\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}$',
    description: 'Credit card number (4111-1111-1111-1111)',
    field_name_indicators: ['credit_card', 'cc_number', 'card_number'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 10
  },

  // Email
  {
    pii_type: 'email',
    pattern: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$',
    description: 'Email address',
    field_name_indicators: ['email', 'email_address', 'contact_email'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 8
  },

  // Phone (US)
  {
    pii_type: 'phone',
    pattern: '^\\(?\\d{3}\\)?[\\s.-]?\\d{3}[\\s.-]?\\d{4}$',
    description: 'US phone number',
    field_name_indicators: ['phone', 'phone_number', 'mobile', 'tel'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 8
  },

  // IP Address
  {
    pii_type: 'ip_address',
    pattern: '^\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}$',
    description: 'IPv4 address',
    field_name_indicators: ['ip', 'ip_address', 'client_ip'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 7
  },

  // Account Number
  {
    pii_type: 'account_number',
    pattern: '^\\d{10,18}$',
    description: 'Bank account number',
    field_name_indicators: ['account', 'account_number', 'bank_account'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 9
  },

  // API Key
  {
    pii_type: 'api_key',
    pattern: '^[a-zA-Z0-9_]{20,}$',
    description: 'API key or secret',
    field_name_indicators: ['api_key', 'secret', 'token', 'key'],
    default_strategy: MaskingStrategy.FULL,
    priority: 10
  }
];
```

### Semantic Detection

Analyze field names for PII indicators.

#### Field Name Matching

```typescript
function detectSemantic(fieldName: string): { pii_type?: string; confidence: number } {
  const normalizedField = fieldName.toLowerCase();

  // Exact match
  const exactMatches: Record<string, string> = {
    'ssn': 'ssn',
    'social_security_number': 'ssn',
    'email': 'email',
    'email_address': 'email',
    'phone': 'phone',
    'phone_number': 'phone',
    'credit_card': 'credit_card',
    'cc_number': 'credit_card',
    'account_number': 'account_number',
    'api_key': 'api_key',
    'secret': 'api_key',
    'password': 'password'
  };

  if (exactMatches[normalizedField]) {
    return {
      pii_type: exactMatches[normalizedField],
      confidence: 0.9
    };
  }

  // Partial match
  if (normalizedField.includes('ssn')) return { pii_type: 'ssn', confidence: 0.75 };
  if (normalizedField.includes('email')) return { pii_type: 'email', confidence: 0.75 };
  if (normalizedField.includes('phone')) return { pii_type: 'phone', confidence: 0.75 };
  if (normalizedField.includes('credit') || normalizedField.includes('card')) {
    return { pii_type: 'credit_card', confidence: 0.7 };
  }
  if (normalizedField.includes('account')) return { pii_type: 'account_number', confidence: 0.7 };
  if (normalizedField.includes('key') || normalizedField.includes('secret')) {
    return { pii_type: 'api_key', confidence: 0.7 };
  }

  return { confidence: 0 };
}
```

---

## Masking Strategies

### 1. Partial Masking

Show first/last N characters, mask middle.

#### Algorithm

```typescript
function applyPartialMasking(
  value: string,
  showFirstN: number,
  showLastN: number,
  maskChar: string
): string {
  const length = value.length;

  // If value too short, full mask
  if (length <= showFirstN + showLastN) {
    return maskChar.repeat(length);
  }

  const first = value.substring(0, showFirstN);
  const last = value.substring(length - showLastN);
  const middleLength = length - showFirstN - showLastN;
  const middle = maskChar.repeat(middleLength);

  return first + middle + last;
}
```

#### Examples

```typescript
// SSN: show last 4
applyPartialMasking('123-45-6789', 0, 4, '*'); // "***-**-6789"

// Credit card: show last 4
applyPartialMasking('4111111111111111', 0, 4, '*'); // "************1111"

// Email: show first 1 and domain
applyPartialMasking('john@example.com', 1, 12, '*'); // "j***@example.com"
```

---

### 2. Full Masking

Replace entire value with `[REDACTED]`.

#### Algorithm

```typescript
function applyFullMasking(value: string): string {
  return '[REDACTED]';
}
```

#### Use Cases

- Names
- Passwords
- API secrets
- Sensitive notes

---

### 3. Tokenization

Generate secure token, store mapping in token store.

#### Algorithm

```typescript
async function generateToken(
  value: string,
  piiType: string
): Promise<string> {
  // 1. Check if token exists for this value (deterministic tokenization)
  const existingToken = await this.findTokenByValue(value, piiType);
  if (existingToken) {
    return existingToken;
  }

  // 2. Generate new token
  const token = `tok_${piiType}_${randomBytes(16).toString('hex')}`;

  // 3. Encrypt value
  const encryptedValue = await this.encrypt(value);

  // 4. Store in token table
  await this.pool.query(`
    INSERT INTO pii_tokens (token, value_encrypted, pii_type, field)
    VALUES ($1, $2, $3, $4)
  `, [token, encryptedValue, piiType, '']);

  return token;
}
```

#### Token Format

```
tok_<pii_type>_<random_hex>

Examples:
- tok_ssn_abc123def456
- tok_email_xyz789abc012
- tok_credit_card_123abc456def
```

---

## Pattern Library

### Full Pattern Library (20+ Patterns)

```typescript
const FULL_PATTERN_LIBRARY: PIIPattern[] = [
  // Identity Numbers
  {
    pii_type: 'ssn',
    pattern: '^\\d{3}-\\d{2}-\\d{4}$',
    description: 'US Social Security Number',
    field_name_indicators: ['ssn', 'social_security'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 10
  },
  {
    pii_type: 'passport',
    pattern: '^[A-Z]\\d{8}$',
    description: 'US Passport Number',
    field_name_indicators: ['passport', 'passport_number'],
    default_strategy: MaskingStrategy.FULL,
    priority: 10
  },
  {
    pii_type: 'drivers_license',
    pattern: '^[A-Z]{2}\\d{6,8}$',
    description: "Driver's License",
    field_name_indicators: ['drivers_license', 'dl', 'license'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 10
  },

  // Financial
  {
    pii_type: 'credit_card',
    pattern: '^\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}$',
    description: 'Credit Card Number',
    field_name_indicators: ['credit_card', 'cc_number', 'card'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 10
  },
  {
    pii_type: 'account_number',
    pattern: '^\\d{10,18}$',
    description: 'Bank Account Number',
    field_name_indicators: ['account', 'account_number', 'bank_account'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 9
  },
  {
    pii_type: 'routing_number',
    pattern: '^\\d{9}$',
    description: 'Bank Routing Number',
    field_name_indicators: ['routing', 'routing_number', 'aba'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 9
  },

  // Contact
  {
    pii_type: 'email',
    pattern: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$',
    description: 'Email Address',
    field_name_indicators: ['email', 'email_address'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 8
  },
  {
    pii_type: 'phone',
    pattern: '^\\(?\\d{3}\\)?[\\s.-]?\\d{3}[\\s.-]?\\d{4}$',
    description: 'US Phone Number',
    field_name_indicators: ['phone', 'mobile', 'tel'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 8
  },

  // Network
  {
    pii_type: 'ip_address',
    pattern: '^\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}$',
    description: 'IPv4 Address',
    field_name_indicators: ['ip', 'ip_address', 'client_ip'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 7
  },
  {
    pii_type: 'mac_address',
    pattern: '^([0-9A-Fa-f]{2}[:-]){5}([0-9A-Fa-f]{2})$',
    description: 'MAC Address',
    field_name_indicators: ['mac', 'mac_address'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 7
  },

  // Authentication
  {
    pii_type: 'api_key',
    pattern: '^(sk_|pk_|api_)[a-zA-Z0-9_]{20,}$',
    description: 'API Key',
    field_name_indicators: ['api_key', 'key', 'secret'],
    default_strategy: MaskingStrategy.FULL,
    priority: 10
  },
  {
    pii_type: 'password',
    pattern: '.*',
    description: 'Password (detected by field name only)',
    field_name_indicators: ['password', 'passwd', 'pwd'],
    default_strategy: MaskingStrategy.FULL,
    priority: 10
  },

  // Healthcare
  {
    pii_type: 'mrn',
    pattern: '^MRN-\\d{6,8}$',
    description: 'Medical Record Number',
    field_name_indicators: ['mrn', 'medical_record'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 9
  },
  {
    pii_type: 'health_insurance',
    pattern: '^[A-Z]\\d{8,10}$',
    description: 'Health Insurance ID',
    field_name_indicators: ['insurance_id', 'health_insurance'],
    default_strategy: MaskingStrategy.PARTIAL,
    priority: 9
  }
];
```

---

## Edge Cases

### Edge Case 1: Empty Value

**Scenario**: Observation value is empty string.

**Resolution**:
```typescript
if (!observation.value || observation.value.trim() === '') {
  return {
    ...observation,
    masked: false,
    pii_detected: false
  };
}
```

---

### Edge Case 2: No PII Detected (False Positive)

**Scenario**: Field name suggests PII, but value doesn't match pattern.

**Example**: Field `"ssn"` with value `"N/A"`

**Resolution**:
- Require both pattern AND semantic match for high confidence
- Use confidence threshold (0.7) to filter false positives

---

### Edge Case 3: Multiple PII Types in Single Value

**Scenario**: Value contains multiple PII types (e.g., "Email: john@example.com, Phone: 555-1234")

**Resolution**:
- Detect highest-priority PII type
- Mask entire value
- Log all detected PII types in metadata

---

### Edge Case 4: Very Long Values (>10KB)

**Scenario**: Observation value is very long (e.g., full document text).

**Resolution**:
- Truncate value to 10KB for pattern matching
- Log warning if truncated
- Consider breaking into smaller observations

---

### Edge Case 5: Token Expiration

**Scenario**: Token exists but has expired.

**Resolution**:
```typescript
const query = `
  SELECT * FROM pii_tokens
  WHERE token = $1
    AND (expires_at IS NULL OR expires_at > NOW())
`;

if (result.rows.length === 0) {
  throw new TokenExpiredError('Token expired or not found');
}
```

---

### Edge Case 6: Deterministic Tokenization Collision

**Scenario**: Two different values hash to same token (unlikely but possible).

**Resolution**:
- Use SHA-256 hash + random salt for token generation
- Store original value (encrypted) to detect collisions
- If collision detected, regenerate token with new salt

---

### Edge Case 7: Partial Masking with Short Values

**Scenario**: Value is "123" but `show_last_n = 4`.

**Resolution**:
```typescript
if (value.length <= showFirstN + showLastN) {
  return maskChar.repeat(value.length); // Full mask
}
```

---

## Performance Characteristics

### Latency Targets

| Operation | Input Size | Target Latency (p95) | Notes |
|-----------|-----------|---------------------|-------|
| `maskObservation()` | 1 observation | < 5ms | Single observation |
| `maskBatch()` | 1000 observations | < 100ms | Batch processing |
| `detectPII()` | 1 observation | < 3ms | Detection only |
| `unmaskObservation()` | 1 token | < 10ms | Includes decryption |

### Throughput

- **Observations/sec**: 10K observations/sec (batch)
- **Detections/sec**: 15K detections/sec

### Scalability

| Tokens Stored | Token Retrieval | Notes |
|--------------|----------------|-------|
| < 1M | < 5ms | Indexed lookup |
| 1M - 10M | < 10ms | Indexed lookup |
| > 10M | < 20ms | Consider sharding |

---

## Implementation Notes

### PostgreSQL Implementation

```typescript
import { Pool } from 'pg';
import { randomBytes, createCipheriv, createDecipheriv } from 'crypto';

export class PostgresPIIMasker implements PIIMasker {
  private pool: Pool;
  private patterns: PIIPattern[];
  private encryptionKey: Buffer;

  constructor(pool: Pool, encryptionKey: string) {
    this.pool = pool;
    this.patterns = FULL_PATTERN_LIBRARY;
    this.encryptionKey = Buffer.from(encryptionKey, 'hex');
  }

  async maskObservation(
    observation: Observation,
    options?: MaskingOptions
  ): Promise<MaskedObservation> {
    // Implementation shown in Core Functionality section
    // ...
  }

  async detectPII(observation: Observation): Promise<PIIDetectionResult> {
    // Implementation shown in Core Functionality section
    // ...
  }

  private async encrypt(value: string): Promise<Buffer> {
    const iv = randomBytes(16);
    const cipher = createCipheriv('aes-256-gcm', this.encryptionKey, iv);

    let encrypted = cipher.update(value, 'utf8');
    encrypted = Buffer.concat([encrypted, cipher.final()]);

    const authTag = cipher.getAuthTag();

    // Combine IV + encrypted + authTag
    return Buffer.concat([iv, encrypted, authTag]);
  }

  private async decrypt(encrypted: Buffer): Promise<string> {
    const iv = encrypted.slice(0, 16);
    const authTag = encrypted.slice(encrypted.length - 16);
    const ciphertext = encrypted.slice(16, encrypted.length - 16);

    const decipher = createDecipheriv('aes-256-gcm', this.encryptionKey, iv);
    decipher.setAuthTag(authTag);

    let decrypted = decipher.update(ciphertext);
    decrypted = Buffer.concat([decrypted, decipher.final()]);

    return decrypted.toString('utf8');
  }

  private applyPartialMasking(
    value: string,
    showFirstN: number,
    showLastN: number,
    maskChar: string
  ): string {
    const length = value.length;

    if (length <= showFirstN + showLastN) {
      return maskChar.repeat(length);
    }

    const first = value.substring(0, showFirstN);
    const last = value.substring(length - showLastN);
    const middleLength = length - showFirstN - showLastN;
    const middle = maskChar.repeat(middleLength);

    return first + middle + last;
  }

  private async generateToken(value: string, piiType: string): Promise<string> {
    // Check if token exists
    const hash = createHash('sha256').update(value).digest('hex');
    const existingQuery = `
      SELECT token FROM pii_tokens
      WHERE value_hash = $1
    `;
    const existing = await this.pool.query(existingQuery, [hash]);

    if (existing.rows.length > 0) {
      return existing.rows[0].token;
    }

    // Generate new token
    const token = `tok_${piiType}_${randomBytes(16).toString('hex')}`;
    const encryptedValue = await this.encrypt(value);

    await this.pool.query(`
      INSERT INTO pii_tokens (token, value_encrypted, value_hash, pii_type, field)
      VALUES ($1, $2, $3, $4, $5)
    `, [token, encryptedValue, hash, piiType, '']);

    return token;
  }
}
```

---

## Security Considerations

### 1. Encryption at Rest

**Encrypt Token Store**:
```typescript
// Use AES-256-GCM for token value encryption
const encrypted = await this.encrypt(value);
```

### 2. Access Control

**Restrict Unmasking**:
```typescript
// Check permission before unmasking
const hasPermission = await this.accessControl.checkPermission({
  user_id: authorization.user_id,
  resource: 'pii',
  action: 'unmask'
});

if (!hasPermission) {
  throw new UnauthorizedError('Not authorized to unmask PII');
}
```

### 3. Audit Trail

**Log All Operations**:
```typescript
// Log masking
await this.pool.query(`
  INSERT INTO pii_masking_log (upload_id, field, pii_type, strategy)
  VALUES ($1, $2, $3, $4)
`, [upload_id, field, pii_type, strategy]);

// Log unmasking
await this.pool.query(`
  INSERT INTO pii_unmasking_log (token, user_id, reason, ip_address)
  VALUES ($1, $2, $3, $4)
`, [token, user_id, reason, ip_address]);
```

---

## Integration Patterns

### Pattern 1: Automatic Masking in Parser

```typescript
class MaskingParser implements Parser {
  constructor(
    private parser: Parser,
    private masker: PIIMasker
  ) {}

  async parse(upload: Upload): Promise<Observation[]> {
    // Parse document
    const observations = await this.parser.parse(upload);

    // Mask PII in observations
    const masked = await this.masker.maskBatch(observations, {
      strategy: 'partial',
      show_last_n: 4
    });

    return masked;
  }
}
```

### Pattern 2: Conditional Masking in Export

```typescript
class ExportEngine {
  async exportObservations(
    uploadId: string,
    options: ExportOptions
  ): Promise<string> {
    const observations = await this.getObservations(uploadId);

    // Mask PII if exporting for external party
    if (options.external) {
      const masked = await this.masker.maskBatch(observations, {
        strategy: 'full'
      });

      return this.formatExport(masked);
    }

    return this.formatExport(observations);
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
describe('PIIMasker', () => {
  let masker: PIIMasker;
  let pool: Pool;

  beforeEach(() => {
    pool = new Pool({ /* test config */ });
    masker = new PostgresPIIMasker(pool, TEST_ENCRYPTION_KEY);
  });

  describe('maskObservation()', () => {
    it('should mask SSN with partial strategy', async () => {
      const observation = {
        field: 'ssn',
        value: '123-45-6789',
        upload_id: 'upload_123'
      };

      const masked = await masker.maskObservation(observation, {
        strategy: 'partial',
        show_last_n: 4
      });

      expect(masked.value).toBe('***-**-6789');
      expect(masked.pii_detected).toBe(true);
      expect(masked.pii_type).toBe('ssn');
    });

    it('should not mask non-PII', async () => {
      const observation = {
        field: 'description',
        value: 'Purchase at grocery store',
        upload_id: 'upload_123'
      };

      const masked = await masker.maskObservation(observation);

      expect(masked.value).toBe('Purchase at grocery store');
      expect(masked.pii_detected).toBe(false);
    });
  });

  describe('detectPII()', () => {
    it('should detect SSN via pattern', async () => {
      const observation = {
        field: 'some_field',
        value: '123-45-6789',
        upload_id: 'upload_123'
      };

      const detection = await masker.detectPII(observation);

      expect(detection.pii_detected).toBe(true);
      expect(detection.pii_type).toBe('ssn');
      expect(detection.detection_method).toBe('pattern');
    });

    it('should detect email via semantic', async () => {
      const observation = {
        field: 'email',
        value: 'invalid-email',
        upload_id: 'upload_123'
      };

      const detection = await masker.detectPII(observation);

      expect(detection.pii_detected).toBe(true);
      expect(detection.detection_method).toBe('semantic');
    });
  });

  describe('unmaskObservation()', () => {
    it('should unmask with proper authorization', async () => {
      // Create token
      const observation = {
        field: 'ssn',
        value: '123-45-6789',
        upload_id: 'upload_123'
      };

      const masked = await masker.maskObservation(observation, {
        strategy: 'tokenize'
      });

      // Unmask
      const unmasked = await masker.unmaskObservation(masked.token!, {
        user_id: 'user_123',
        reason: 'Customer service inquiry'
      });

      expect(unmasked.value).toBe('123-45-6789');
    });

    it('should reject unmasking without authorization', async () => {
      await expect(
        masker.unmaskObservation('tok_ssn_abc123', {
          user_id: 'unauthorized_user',
          reason: 'No reason'
        })
      ).rejects.toThrow('Not authorized');
    });
  });
});
```

### Performance Tests

```typescript
describe('PIIMasker Performance', () => {
  it('should mask observation in <5ms', async () => {
    const observation = {
      field: 'ssn',
      value: '123-45-6789',
      upload_id: 'upload_123'
    };

    const startTime = Date.now();
    await masker.maskObservation(observation);
    const duration = Date.now() - startTime;

    console.log(`Masking duration: ${duration}ms`);
    expect(duration).toBeLessThan(5);
  });

  it('should mask 1000 observations in <100ms', async () => {
    const observations = Array.from({ length: 1000 }, (_, i) => ({
      field: 'ssn',
      value: `123-45-${String(i).padStart(4, '0')}`,
      upload_id: 'upload_123'
    }));

    const startTime = Date.now();
    await masker.maskBatch(observations);
    const duration = Date.now() - startTime;

    console.log(`Batch masking duration: ${duration}ms`);
    expect(duration).toBeLessThan(100);
  });
});
```

---

## Configuration

### Tunable Parameters

```typescript
interface PIIMaskerConfig {
  // Detection
  detection_threshold: number;      // Default: 0.7
  patterns: PIIPattern[];           // Custom patterns

  // Masking
  default_strategy: MaskingStrategy; // Default: "partial"
  default_show_last_n: number;      // Default: 4
  default_mask_char: string;        // Default: "*"

  // Token Store
  token_expiration_days: number;    // Default: 365
  token_cleanup_interval_hours: number; // Default: 24

  // Performance
  batch_size: number;               // Default: 1000
  encryption_algorithm: string;     // Default: "aes-256-gcm"
}
```

---

## Observability

### Metrics

```typescript
interface PIIMaskerMetrics {
  observations_masked_total: Counter;
  pii_detected_total: Counter;
  pii_detection_rate: Gauge;
  masking_latency_ms: Histogram;
  unmasking_requests_total: Counter;
  unmasking_errors_total: Counter;
}
```

---

## Future Enhancements

### Phase 1: ML-Based Detection (3 months)
- Train ML model for PII detection
- Reduce false positives
- Detect new PII types automatically

### Phase 2: Format-Preserving Encryption (6 months)
- Maintain data format after masking
- Example: Masked SSN still matches SSN format

### Phase 3: Differential Privacy (12 months)
- Add noise to sensitive numeric values
- Preserve statistical properties while protecting privacy

---

**End of PIIMasker OL Primitive Specification**
