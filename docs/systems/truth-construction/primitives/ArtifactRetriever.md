# ArtifactRetriever (OL Primitive)

## Definition

**ArtifactRetriever** retrieves original upload artifacts (PDFs, images, CSVs, videos, etc.) from StorageEngine with access control and signed URLs. It handles streaming for large files, deletion tracking, and expiring URLs for security.

**Problem it solves:**
- Direct access to StorageEngine bypasses access control (security risk)
- Large files (>50MB) can't be sent in single HTTP response (memory overflow)
- Permanent download URLs enable unauthorized sharing
- Deleted artifacts need graceful handling (show deletion notice, not 404)

**Solution:**
- Generates signed, expiring URLs (HMAC-SHA256)
- Streams large files in chunks (no memory spike)
- Enforces access control (user owns artifact)
- Tracks artifact deletion (retention policy compliance)
- Supports inline view (PDF viewer) vs download

---

## Simplicity Profiles

**This section shows how the same universal retrieval primitive is configured differently based on usage scale.**

### Profile 1: Personal Use (Finance App - Single User, Direct Access)

**Context:** Darwin accesses his own uploaded bank statements (single user, no sharing)

**Configuration:**
```yaml
artifact_retriever:
  signed_urls: false  # Single user, no need to sign URLs
  access_control: false  # Single user owns all uploads
  streaming: false  # Files <10MB (fit in memory)
  url_expiration_seconds: null  # URLs don't expire
```

**What's Used:**
- ✅ Basic metadata retrieval - `get_metadata(upload_id)` returns filename, size, mime_type
- ✅ Direct file retrieval - Read file from StorageEngine, return bytes
- ✅ Inline vs download - Support PDF inline view vs download
- ✅ Deletion tracking - Show "This statement was deleted" if deleted

**What's NOT Used:**
- ❌ Signed URLs - Single user, no security risk
- ❌ URL expiration - No unauthorized sharing risk
- ❌ Access control checks - Single user owns all files
- ❌ Streaming - Files <10MB (fit in memory)
- ❌ HMAC signature verification - No signed URLs
- ❌ Audit logging - No compliance requirement

**Implementation Complexity:** **LOW**
- ~80 lines Python
- Simple endpoint: `GET /uploads/{upload_id}/download`
- Reads file from StorageEngine, streams HTTP response
- No access control checks (single user app)

**Narrative Example:**
> Darwin clicks "View Statement" for Oct 2024 upload. UI calls `/uploads/UL_abc123/download`. ArtifactRetriever retrieves metadata → `{"filename": "BoFA_Oct2024.pdf", "mime_type": "application/pdf", "size_bytes": 2300000}`. Retrieve file from StorageEngine → Read 2.3MB into memory → Return HTTP response with headers `Content-Type: application/pdf`, `Content-Disposition: inline` (opens in browser PDF viewer). If Darwin clicks "Download" → Same process, but `Content-Disposition: attachment; filename="BoFA_Oct2024.pdf"` (triggers browser download).

---

### Profile 2: Small Business (Accounting Firm - Multi-User, Basic Security)

**Context:** 10 accountants access client uploads, need basic access control (accountant A can't access accountant B's files)

**Configuration:**
```yaml
artifact_retriever:
  signed_urls: true  # Generate secure URLs
  url_expiration_seconds: 3600  # 1 hour expiration
  access_control: true  # Check user owns upload
  streaming:
    enabled: true
    threshold_mb: 10  # Stream files >10MB
  signing_secret: "${SIGNING_SECRET}"  # HMAC key
```

**What's Used:**
- ✅ Signed URLs - Generate URLs with HMAC-SHA256 signature
- ✅ URL expiration - 1-hour expiration (prevent link sharing)
- ✅ Access control - Check `upload.uploaded_by == user_id`
- ✅ Streaming - Files >10MB streamed in 8KB chunks
- ✅ Metadata retrieval with access check
- ✅ Deletion tracking

**What's NOT Used:**
- ❌ Audit logging - Just basic access control (no compliance)
- ❌ CDN integration - Direct download from app server
- ❌ Rate limiting - Small user base (10 users)

**Implementation Complexity:** **MEDIUM**
- ~250 lines Python
- HMAC-SHA256 URL signing
- Access control middleware
- Streaming for large files
- URL format: `/uploads/{upload_id}/download?sig=a3b5c7...&exp=1730000000`

**Narrative Example:**
> Accountant Alice clicks "View Receipt" for client upload. UI calls `/api/generate-download-url` with `upload_id="UL_123"`. ArtifactRetriever checks: Does Alice own this upload? Query: `SELECT uploaded_by FROM upload_records WHERE upload_id='UL_123'` → `uploaded_by='user_alice'` ✅ Match. Generate signed URL: `base_url="/uploads/UL_123/download" + timestamp=1730000000 + signature=HMAC_SHA256(signing_secret, "UL_123|1730000000")` → Returns `{"url": "/uploads/UL_123/download?sig=a3b5c7&exp=1730000000", "expires_at": "2024-10-27T11:00:00Z"}`. Alice opens URL → ArtifactRetriever verifies signature → Valid ✅ → Check expiration → `now() < 1730000000` ✅ → Stream file (PDF 450KB, <10MB threshold, loaded into memory) → Return HTTP response.
>
> Alternative: Accountant Bob tries to access Alice's upload URL → Copy-paste signed URL → ArtifactRetriever verifies signature ✅ (valid), checks expiration ✅ (within 1 hour), checks access → Query: `uploaded_by='user_alice'` ≠ `user_bob` ❌ → Return `403 Forbidden: "You don't own this upload"`.

---

### Profile 3: Enterprise (Bank - High Security, Audit, CDN)

**Context:** Bank serves 10,000 uploads/day to 1M customers, strict security (GLBA compliance), serve via CDN for performance

**Configuration:**
```yaml
artifact_retriever:
  signed_urls: true
  url_expiration_seconds: 300  # 5 minutes (short-lived for security)
  access_control:
    enabled: true
    method: "database"  # Query access control table
    cache_ttl_seconds: 60  # Cache access decisions
  streaming:
    enabled: true
    threshold_mb: 1  # Always stream (even 1MB files, for consistency)
    chunk_size_kb: 64  # Larger chunks for CDN efficiency
  signing:
    algorithm: "hmac-sha256"
    secret_rotation: true  # Rotate signing keys monthly
    secret_version_tracking: true
  cdn:
    enabled: true
    provider: "cloudfront"
    signed_cookies: true  # CloudFront signed cookies
  audit:
    log_every_access: true
    log_to: "audit_log_table"
    include_fields: ["user_id", "upload_id", "ip_address", "user_agent"]
  rate_limiting:
    enabled: true
    max_requests_per_minute: 60  # Per user
  metrics:
    enabled: true
    backend: "prometheus"
    labels: ["document_type", "access_result"]
```

**What's Used:**
- ✅ HMAC-signed URLs with short expiration (5 minutes)
- ✅ Secret rotation - Monthly key rotation for security
- ✅ Access control with caching - Cache decisions for 60s (reduce DB load)
- ✅ Always streaming - Even 1MB files (consistency)
- ✅ CDN integration - CloudFront signed cookies
- ✅ Audit logging - Log every access attempt
- ✅ Rate limiting - 60 requests/min per user (prevent abuse)
- ✅ Prometheus metrics - Track access patterns, latencies

**What's NOT Used:**
- (All features enabled at enterprise scale)

**Implementation Complexity:** **HIGH**
- ~900 lines TypeScript/Python
- HMAC signing with key rotation
- CloudFront signed cookie generation
- Access control with Redis caching
- Comprehensive audit logging
- Rate limiting (Redis-backed)
- Monitoring dashboards

**Narrative Example:**
> Customer logs into bank portal, clicks "Download October Statement" → UI calls `/api/generate-download-url` with `upload_id="UL_stmt_456"`. ArtifactRetriever:
> 1. Access control check (cached): Redis lookup `access:user_12345:UL_stmt_456` → Cache miss → Query DB: `SELECT customer_id FROM upload_records WHERE upload_id='UL_stmt_456'` → `customer_id='12345'` ✅ → Cache result in Redis (TTL=60s)
> 2. Rate limit check: Redis INCR `ratelimit:user_12345:minute` → Count=15 (< 60 limit) ✅
> 3. Generate CloudFront signed cookie: `Policy=base64({"Resource":"https://cdn.bank.com/uploads/*","Condition":{"DateLessThan":{"AWS:EpochTime":1730000300}}})` + `Signature=RSA_SHA256(cloudfront_private_key, policy)`
> 4. Generate backend signed URL: `url="/uploads/UL_stmt_456/download"`, `timestamp=now()+300`, `signature=HMAC_SHA256(signing_secret_v3, "UL_stmt_456|1730000300")` → Returns `{"url": "https://cdn.bank.com/uploads/UL_stmt_456/download?sig=a3b5c7&exp=1730000300&v=3", "expires_at": "2024-10-27T10:05:00Z", "signed_cookie": "CloudFront-Policy=...; CloudFront-Signature=..."}`
> 5. Audit log: `INSERT INTO audit_log (event="download_url_generated", user_id="12345", upload_id="UL_stmt_456", ip_address="203.0.113.45", timestamp=now())`
> 6. Prometheus metric: `download_url_generations_total{document_type="statement",access_result="success"}++`
>
> Customer clicks download link → CloudFront validates signed cookie → Valid ✅ → CloudFront requests file from origin (app server) → ArtifactRetriever verifies backend signature → Valid ✅, not expired ✅ → Stream file from S3 (2.5MB, 64KB chunks) → CloudFront caches response → Serve to customer (fast, low latency). Total time: <500ms (CDN cached).
>
> 5 minutes later, customer shares URL with friend → Friend opens URL → CloudFront validates cookie → Expired ❌ → Return `403 Forbidden: "Download link expired"`.

---

**Key Insight:** The same `ArtifactRetriever` interface works across all 3 profiles. Personal use has direct access (no security needed); Small business adds signed URLs and basic access control; Enterprise adds short expiration, CDN integration, audit logs, and rate limiting for compliance and performance.

---

## Interface Contract

```python
from typing import Optional, Generator
from dataclasses import dataclass
import hashlib
import hmac
import time

@dataclass
class ArtifactMetadata:
    """Artifact metadata (without file content)"""
    upload_id: str
    filename: str
    hash: str  # SHA-256
    size_bytes: int
    mime_type: str
    uploaded_by: str
    uploaded_at: str  # ISO 8601
    source_type: str
    status: str  # "available" | "deleted"
    deleted_at: Optional[str]  # ISO 8601 (if deleted)

@dataclass
class SignedURL:
    """Signed URL for artifact access"""
    url: str
    expires_at: str  # ISO 8601
    content_disposition: str  # "inline" | "attachment"

class ArtifactRetriever:
    """
    Retrieves upload artifacts from StorageEngine with access control.

    Domain-agnostic - works on ANY file type.
    """

    def __init__(
        self,
        storage_engine,
        upload_record_store,
        signing_secret: str
    ):
        """
        Initialize with storage engine and signing secret.

        Args:
            storage_engine: Source of artifact files
            upload_record_store: Source of upload metadata (for access control)
            signing_secret: Secret key for URL signing (HMAC-SHA256)
        """

    def get_metadata(
        self,
        upload_id: str,
        user_id: str
    ) -> ArtifactMetadata:
        """
        Retrieves artifact metadata (without file content).

        Args:
            upload_id: Upload ID
            user_id: Current user (for access control)

        Returns:
            ArtifactMetadata

        Raises:
            ArtifactNotFoundError: If artifact doesn't exist
            UnauthorizedError: If user doesn't own this artifact
        """

    def generate_signed_url(
        self,
        upload_id: str,
        user_id: str,
        expires_in_seconds: int = 3600,
        content_disposition: str = "inline"
    ) -> SignedURL:
        """
        Generates signed URL for artifact access.

        Args:
            upload_id: Upload ID
            user_id: Current user (for access control)
            expires_in_seconds: URL expiration (default: 1 hour)
            content_disposition: "inline" (PDF viewer) or "attachment" (download)

        Returns:
            SignedURL with expiring URL

        Raises:
            ArtifactNotFoundError: If artifact doesn't exist
            ArtifactDeletedError: If artifact was deleted
        """

    def stream_artifact(
        self,
        upload_id: str,
        user_id: str,
        chunk_size: int = 8192
    ) -> Generator[bytes, None, None]:
        """
        Streams artifact content in chunks (for large files).

        Args:
            upload_id: Upload ID
            user_id: Current user (for access control)
            chunk_size: Chunk size in bytes (default: 8KB)

        Yields:
            bytes chunks

        Raises:
            ArtifactNotFoundError: If artifact doesn't exist
            ArtifactDeletedError: If artifact was deleted

        Example:
            for chunk in retriever.stream_artifact("upl_123", "usr_eugenio"):
                response.write(chunk)
        """

    def verify_signed_url(
        self,
        url: str,
        signature: str,
        timestamp: int
    ) -> bool:
        """
        Verifies signed URL is valid and not expired.

        Args:
            url: Base URL (without signature)
            signature: HMAC-SHA256 signature
            timestamp: Unix timestamp when URL was signed

        Returns:
            True if valid, False otherwise
        """

    def _sign_url(
        self,
        upload_id: str,
        expires_at: int
    ) -> str:
        """
        Internal: Generates HMAC-SHA256 signature for URL.

        Args:
            upload_id: Upload ID
            expires_at: Unix timestamp when URL expires

        Returns:
            HMAC signature (hex string)
        """
```

---

## Multi-Domain Applicability

**Finance:**
```python
retriever = ArtifactRetriever(...)
url = retriever.generate_signed_url("upl_bofa_123", "usr_eugenio")
# Returns: https://api.example.com/uploads/upl_bofa_123/artifact?sig=a3b5c7...&exp=1716480000
# Opens: Bank statement PDF
```

**Healthcare:**
```python
retriever = ArtifactRetriever(...)
url = retriever.generate_signed_url("upl_lab_456", "usr_patient_789")
# Returns: Signed URL for lab report PDF
```

**Legal:**
```python
retriever = ArtifactRetriever(...)
url = retriever.generate_signed_url("upl_contract_789", "usr_lawyer_01")
# Returns: Signed URL for contract PDF
```

**Research:**
```python
retriever = ArtifactRetriever(...)
url = retriever.generate_signed_url("upl_paper_202", "usr_researcher_88")
# Returns: Signed URL for research paper PDF
```

**Manufacturing:**
```python
retriever = ArtifactRetriever(...)
url = retriever.generate_signed_url("upl_inspection_567", "usr_inspector_12")
# Returns: Signed URL for inspection report
```

**Media:**
```python
retriever = ArtifactRetriever(...)
url = retriever.generate_signed_url("upl_video_999", "usr_editor_55")
# Returns: Signed URL for video file (MP4)
```

**Domain-agnostic nature:** Works on ANY file type. Doesn't care if it's PDF, CSV, image, or video.

---

## Responsibilities

**ArtifactRetriever IS responsible for:**
- ✅ Retrieving artifact metadata (filename, size, mime type)
- ✅ Generating signed, expiring URLs
- ✅ Streaming large files in chunks
- ✅ Enforcing access control (user owns artifact)
- ✅ Tracking artifact deletion (retention policy)
- ✅ Supporting inline view vs download modes

**ArtifactRetriever is NOT responsible for:**
- ❌ Storing artifacts (that's StorageEngine)
- ❌ Uploading files (that's UploadRecord)
- ❌ Parsing artifacts (that's Parser)
- ❌ Rendering UI viewers (that's IL layer)
- ❌ Implementing retention policies (that's scheduled job)

---

## Implementation Notes

### URL Signing (HMAC-SHA256)

```python
import hmac
import hashlib
import time

def _sign_url(upload_id, expires_at, signing_secret):
    """Generate HMAC-SHA256 signature for URL"""
    message = f"{upload_id}:{expires_at}"
    signature = hmac.new(
        signing_secret.encode(),
        message.encode(),
        hashlib.sha256
    ).hexdigest()
    return signature

def generate_signed_url(upload_id, user_id, expires_in_seconds=3600):
    # Verify access
    artifact = storage_engine.get_metadata(upload_id)
    if artifact.uploaded_by != user_id:
        raise UnauthorizedError(f"User {user_id} doesn't own {upload_id}")

    # Generate signed URL
    expires_at = int(time.time()) + expires_in_seconds
    signature = _sign_url(upload_id, expires_at, signing_secret)

    url = f"https://api.example.com/uploads/{upload_id}/artifact?sig={signature}&exp={expires_at}"

    return SignedURL(
        url=url,
        expires_at=datetime.fromtimestamp(expires_at).isoformat(),
        content_disposition="inline"
    )
```

### Streaming Large Files

```python
def stream_artifact(upload_id, user_id, chunk_size=8192):
    """Stream artifact in chunks to avoid memory overflow"""
    # Verify access
    artifact = storage_engine.get_metadata(upload_id)
    if artifact.uploaded_by != user_id:
        raise UnauthorizedError(...)

    # Stream from storage
    file_path = storage_engine.get_path(upload_id)
    with open(file_path, 'rb') as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            yield chunk
```

### Handling Deleted Artifacts

```python
def get_metadata(upload_id, user_id):
    artifact = storage_engine.get_metadata(upload_id)

    # Verify access
    if artifact.uploaded_by != user_id:
        raise UnauthorizedError(...)

    # Artifact may be deleted (retention policy)
    if artifact.status == "deleted":
        return ArtifactMetadata(
            upload_id=upload_id,
            filename=artifact.filename,
            hash=artifact.hash,  # Hash preserved for audit
            size_bytes=0,
            mime_type=artifact.mime_type,
            uploaded_by=artifact.uploaded_by,
            uploaded_at=artifact.uploaded_at,
            source_type=artifact.source_type,
            status="deleted",
            deleted_at=artifact.deleted_at
        )

    return artifact
```

### Content Disposition

```python
# Inline (PDF viewer embeds file)
url = retriever.generate_signed_url("upl_123", "usr_eugenio", content_disposition="inline")
# Header: Content-Disposition: inline; filename="statement.pdf"

# Attachment (browser downloads file)
url = retriever.generate_signed_url("upl_123", "usr_eugenio", content_disposition="attachment")
# Header: Content-Disposition: attachment; filename="statement.pdf"
```

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Retrieve uploaded bank statements with access control and signed URLs
**Example:** User requests statement download → Frontend calls GET /uploads/upl_123/download → ArtifactRetriever.generate_signed_url("upl_123", "usr_jane", ttl_seconds=3600) → Verifies access (usr_jane owns upload) → Returns signed URL "https://storage.example.com/artifacts/abc123.pdf?signature=...&expires=..." (expires in 1 hour) → User opens URL → ArtifactRetriever.stream_artifact() streams 2.5MB PDF in 8KB chunks → Browser displays PDF → URL expires after 1 hour (signature invalid, prevents unauthorized sharing)
**Operations:** generate_signed_url (HMAC-SHA256 signature with expiry), stream_artifact (chunked streaming for large files), verify_access (enforce ownership), track_deletion (show "Artifact deleted" instead of 404)
**Performance:** <10ms URL generation, streaming 2.5MB in <100ms
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Retrieve medical images (DICOM files) with HIPAA-compliant access control
**Example:** Radiologist requests X-ray → ArtifactRetriever verifies access (radiologist assigned to patient) → Generates signed URL (1-hour expiry) → Streams 15MB DICOM file → URL expires automatically (HIPAA security requirement: no permanent URLs to PHI)
**Status:** ✅ Conceptually validated

### ✅ Legal
**Use case:** Retrieve signed contracts with chain of custody tracking
**Example:** Attorney downloads contract → ArtifactRetriever verifies access → Generates signed URL (4-hour expiry for document review) → Streams 500KB PDF → Logs download event to ProvenanceLedger (audit trail for discovery requests)
**Status:** ✅ Conceptually validated

### ✅ RSRCH (Utilitario Research)
**Use case:** Retrieve original scraped web pages (HTML archives) for fact verification
**Example:** Analyst reviews TechCrunch article source → Requests original HTML → ArtifactRetriever.get_metadata("upl_tc_789") → Returns {filename: "techcrunch_2024-02-25.html", size: 120KB, status: "available"} → generate_signed_url() creates expiring link → stream_artifact() returns HTML → Analyst verifies extracted facts match source → Later, source deleted (retention policy: 90 days) → get_metadata() returns {status: "deleted", deleted_at: "2024-05-25"} → UI shows "Source archived (retention policy)" instead of 404
**Operations:** Deletion tracking critical (show when/why source deleted, not broken link), signed URLs prevent unauthorized sharing of proprietary research sources, streaming for large podcast transcripts (200KB-2MB)
**Status:** ✅ Conceptually validated

### ✅ E-commerce
**Use case:** Retrieve supplier catalog files with vendor access control
**Example:** Merchant downloads supplier CSV (1.5MB, 1500 products) → ArtifactRetriever verifies ownership → Generates signed URL (24-hour expiry for bulk processing) → Streams CSV in chunks → Merchant processes catalog → URL expires after 24 hours
**Status:** ✅ Conceptually validated

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (signed URL generation, streaming, access control are universal patterns, no domain-specific code)
**Reusability:** High (same generate_signed_url/stream_artifact/verify_access operations work for PDFs, DICOM files, contracts, HTML archives, CSV files; only MIME types and retention policies differ)

---

## Related Primitives

**Upstream dependencies:**
- **StorageEngine** (1.1) - Source of artifact files and metadata
- **UploadRecord** (1.1) - Source of upload metadata (for access control)

**Downstream consumers:**
- **ProvenanceTracer** (2.2) - Uses ArtifactRetriever to generate download URLs
- **DrillDownPanel** (IL) - Displays artifact viewer using signed URLs
- **ArtifactViewer** (IL) - Embeds PDF/image using signed URLs

**Composition pattern:**
```python
# ArtifactRetriever used by ProvenanceTracer
class ProvenanceTracer:
    def __init__(self, ..., artifact_retriever):
        self.artifact = artifact_retriever

    def trace_lineage(self, canonical_id, user_id):
        ...
        # Get artifact download URL
        artifact_url = self.artifact.generate_signed_url(
            upload_id=canonical.upload_id,
            user_id=user_id
        )

        return DrillDownData(
            artifact={
                "download_url": artifact_url.url,
                "expires_at": artifact_url.expires_at,
                ...
            }
        )
```

---

## Example Usage

### Generate Signed URL (Inline View)

```python
from primitives.ol import ArtifactRetriever

# Initialize
retriever = ArtifactRetriever(
    storage_engine=storage_engine,
    upload_record_store=upload_record_store,
    signing_secret="secret_key_abc123"
)

# Generate signed URL for PDF viewer
signed_url = retriever.generate_signed_url(
    upload_id="upl_bofa_20250426",
    user_id="usr_eugenio",
    expires_in_seconds=3600,  # 1 hour
    content_disposition="inline"
)

print(signed_url.url)
# https://api.example.com/uploads/upl_bofa_20250426/artifact?sig=a3b5c7d9...&exp=1716480000

print(signed_url.expires_at)
# 2025-05-23T15:30:00Z
```

### Generate Download URL

```python
# Force download instead of inline view
download_url = retriever.generate_signed_url(
    upload_id="upl_bofa_20250426",
    user_id="usr_eugenio",
    content_disposition="attachment"  # Downloads instead of viewing
)
```

### Stream Large File

```python
# Stream artifact in chunks (for large PDFs)
def download_artifact(upload_id, user_id):
    # Generate response headers
    metadata = retriever.get_metadata(upload_id, user_id)
    headers = {
        "Content-Type": metadata.mime_type,
        "Content-Length": metadata.size_bytes,
        "Content-Disposition": f'attachment; filename="{metadata.filename}"'
    }

    # Stream chunks
    def generate():
        for chunk in retriever.stream_artifact(upload_id, user_id):
            yield chunk

    return StreamingResponse(generate(), headers=headers)
```

### Handle Deleted Artifact

```python
try:
    signed_url = retriever.generate_signed_url("upl_deleted_123", "usr_eugenio")
except ArtifactDeletedError as e:
    print(f"Artifact deleted on {e.deleted_at}")
    print(f"Hash for audit: {e.hash}")
    # Show UI message: "File deleted (retention policy). Hash: abc123... available for verification."
```

### Healthcare Example (Lab Report)

```python
# Same ArtifactRetriever, different domain
signed_url = retriever.generate_signed_url(
    upload_id="upl_lab_report_456",
    user_id="usr_patient_789"
)
# Returns signed URL for lab_report_2025-04-15.pdf
```

**Key insight:** Same ArtifactRetriever for all file types. It doesn't care if it's a bank statement, lab report, or contract - it just retrieves files securely.
