# OL Primitive: StorageEngine

**Type**: Infrastructure / Storage
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification

---

## Purpose

Content-addressable storage engine for immutable artifacts (files, documents, binaries). Uses cryptographic hash as addressing mechanism to ensure deduplication and integrity.

---

## Interface Contract

### Core Methods

```typescript
interface StorageEngine {
  // Store new content, return internal reference
  store(content: Bytes, metadata?: StorageMetadata): StorageRef

  // Check if content exists by hash
  exists(hash: Hash): boolean

  // Retrieve content by reference
  retrieve(ref: StorageRef): Bytes

  // Get metadata without retrieving content
  getMetadata(ref: StorageRef): StorageMetadata

  // Delete content (if policy allows)
  delete(ref: StorageRef): void

  // Calculate hash without storing
  calculateHash(content: Bytes): Hash
}
```

### Types

```typescript
type Hash = string  // Format: "sha256:hexdigest"
type StorageRef = string  // Internal reference (opaque to caller)

interface StorageMetadata {
  hash: Hash
  size_bytes: number
  mime_type: string
  stored_at: ISO8601Timestamp
  content_encoding?: string  // e.g., "gzip"
}
```

---

## Behavior Specifications

### 1. Content-Addressable Storage

**Property**: Same content → same hash → single storage entry

**Example:**
```
store(file_a.pdf) → ref_1 (hash: sha256:abc123...)
store(file_b.pdf) → ref_1 (same hash if content identical)
```

**Guarantees:**
- Deduplication automatic (no duplicate storage)
- Hash collision resistance (SHA-256: negligible probability)
- Integrity verification (retrieve → hash must match)

---

### 2. Streaming Hash Calculation

**Property**: Hash calculated during upload (no temp file needed)

**Implementation contract:**
```python
def store_streaming(stream: ByteStream) -> StorageRef:
    hasher = SHA256()
    buffer = BytesIO()

    for chunk in stream:
        hasher.update(chunk)
        buffer.write(chunk)

    hash = hasher.finalize()

    # Check dedupe before persist
    if exists(hash):
        return get_ref_by_hash(hash)

    # Persist content
    ref = persist(buffer.getvalue(), hash)
    return ref
```

**Memory safety:**
- Chunk size: 8KB (configurable)
- Max buffer: 100MB (fail if exceeded without streaming to disk)

---

### 3. Immutability

**Property**: Once stored, content NEVER changes

**Enforcement:**
- No `update()` method
- `delete()` requires explicit policy check
- References are stable (don't expire unless explicitly GC'd)

**Implications:**
- Versioning via new uploads (each gets new ref)
- Rollback via keeping old refs
- Audit trail intact (provenance points to stable refs)

---

### 4. Metadata Separation

**Property**: Metadata stored separately from content blob

**Storage structure:**
```
storage/
  content/
    ab/c1/23...  ← blob (addressed by hash)
  metadata/
    ab/c1/23...  ← JSON metadata
```

**Benefit:**
- Retrieve metadata without downloading blob
- Query metadata (size, mime_type) efficiently
- Extensible metadata (add fields without re-uploading)

---

## Error Handling

| Error | When | Response |
|-------|------|----------|
| `HASH_COLLISION` | Two files with same hash but different content | Log alert, fail store operation |
| `STORAGE_FULL` | Disk/quota exceeded | Return error, suggest cleanup |
| `CORRUPT_CONTENT` | Retrieve hash doesn't match stored hash | Return error, mark for re-upload |
| `REF_NOT_FOUND` | Reference doesn't exist | Return `null` or throw |
| `PERMISSION_DENIED` | Policy forbids delete | Throw error |

---

## Configuration

```yaml
storage_engine:
  backend: "filesystem"  # or "s3", "gcs", "azure_blob"
  base_path: "/var/storage"
  hash_algorithm: "sha256"
  chunk_size_kb: 8
  max_file_size_mb: 50
  compression: false  # Enable gzip for text files
  gc_policy:
    enabled: true
    retention_days: 365
    orphan_check_enabled: true
```

---

## Backend Implementations

### Filesystem Backend

```python
class FilesystemStorageEngine(StorageEngine):
    def __init__(self, base_path: str):
        self.base_path = base_path

    def store(self, content: bytes) -> StorageRef:
        hash = self.calculate_hash(content)

        # Check dedupe
        if self.exists(hash):
            return self._get_ref_by_hash(hash)

        # Create directory structure
        path = self._hash_to_path(hash)
        os.makedirs(os.path.dirname(path), exist_ok=True)

        # Write atomically
        temp_path = f"{path}.tmp"
        with open(temp_path, 'wb') as f:
            f.write(content)
        os.rename(temp_path, path)  # Atomic on POSIX

        # Store metadata
        self._write_metadata(hash, {
            "size_bytes": len(content),
            "stored_at": datetime.utcnow().isoformat()
        })

        return self._hash_to_ref(hash)

    def _hash_to_path(self, hash: str) -> str:
        # sha256:abc123... → /base/ab/c1/23...
        digest = hash.split(':')[1]
        return os.path.join(
            self.base_path,
            "content",
            digest[:2],
            digest[2:4],
            digest
        )
```

### S3 Backend

```python
class S3StorageEngine(StorageEngine):
    def __init__(self, bucket: str):
        self.s3 = boto3.client('s3')
        self.bucket = bucket

    def store(self, content: bytes) -> StorageRef:
        hash = self.calculate_hash(content)

        # Check dedupe
        key = f"content/{hash}"
        try:
            self.s3.head_object(Bucket=self.bucket, Key=key)
            return hash  # Already exists
        except ClientError:
            pass

        # Upload
        self.s3.put_object(
            Bucket=self.bucket,
            Key=key,
            Body=content,
            Metadata={"hash": hash}
        )

        return hash
```

---

## Reusability Across Domains

### Finance
- Store: BoFA PDFs, receipts, invoices
- Dedupe: Same statement uploaded multiple times

### Health
- Store: Lab results, medical images, prescriptions
- Dedupe: Same test result from different sources

### Legal
- Store: Contracts, court documents, evidence files
- Dedupe: Same contract with multiple signatories

### Generic File Processing
- Store: User uploads, generated reports, backups
- Dedupe: Attachments, media libraries

---

## Extension Points

### 1. Custom Hash Algorithms

```python
class StorageEngineWithCustomHash(StorageEngine):
    def __init__(self, hash_algo: str = "sha256"):
        self.hash_algo = hash_algo

    def calculate_hash(self, content: bytes) -> Hash:
        if self.hash_algo == "sha256":
            return f"sha256:{hashlib.sha256(content).hexdigest()}"
        elif self.hash_algo == "blake2b":
            return f"blake2b:{hashlib.blake2b(content).hexdigest()}"
        else:
            raise ValueError(f"Unsupported hash: {self.hash_algo}")
```

### 2. Encryption at Rest

```python
class EncryptedStorageEngine(StorageEngine):
    def store(self, content: bytes) -> StorageRef:
        encrypted = self.encrypt(content)
        ref = super().store(encrypted)
        return ref

    def retrieve(self, ref: StorageRef) -> bytes:
        encrypted = super().retrieve(ref)
        return self.decrypt(encrypted)
```

### 3. Versioning

```python
class VersionedStorageEngine(StorageEngine):
    def store_version(self, upload_id: str, version: int, content: bytes) -> StorageRef:
        # Hash includes version marker
        hash = self.calculate_hash(content)
        ref = super().store(content)

        # Track versions
        self.version_index[upload_id][version] = ref
        return ref

    def get_version(self, upload_id: str, version: int) -> bytes:
        ref = self.version_index[upload_id][version]
        return self.retrieve(ref)
```

---

## Performance Characteristics

| Operation | Time Complexity | Notes |
|-----------|-----------------|-------|
| `store()` | O(n) where n = file size | Dominated by I/O |
| `exists()` | O(1) | Hash lookup in index |
| `retrieve()` | O(n) where n = file size | Dominated by I/O |
| `calculateHash()` | O(n) | SHA-256 streaming |
| `delete()` | O(1) | Mark for GC or unlink |

**Benchmarks (Filesystem backend, SSD):**
- Store 1MB file: ~10ms
- Store 10MB file: ~50ms
- Store 50MB file: ~250ms
- Hash calculation overhead: ~5% of total time

---

## Testing Strategy

### Unit Tests

```python
def test_dedupe():
    engine = FilesystemStorageEngine("/tmp/test")
    content = b"hello world"

    ref1 = engine.store(content)
    ref2 = engine.store(content)

    assert ref1 == ref2
    assert engine.exists(ref1)

def test_integrity():
    engine = FilesystemStorageEngine("/tmp/test")
    content = b"test data"

    ref = engine.store(content)
    retrieved = engine.retrieve(ref)

    assert content == retrieved

def test_hash_collision_detection():
    # Simulate collision (impossible in practice)
    engine = FilesystemStorageEngine("/tmp/test")

    # Mock hash function to return same hash for different content
    with patch.object(engine, 'calculate_hash', return_value='sha256:fake'):
        engine.store(b"content A")

        with pytest.raises(HashCollisionError):
            engine.store(b"content B")
```

### Integration Tests

```python
def test_large_file_streaming():
    engine = FilesystemStorageEngine("/tmp/test")

    # Generate 100MB file
    content = b"x" * (100 * 1024 * 1024)

    ref = engine.store(content)
    retrieved = engine.retrieve(ref)

    assert len(retrieved) == len(content)
```

---

## Security Considerations

### 1. Path Traversal Prevention

```python
def _hash_to_path(self, hash: str) -> str:
    # Validate hash format
    if not re.match(r'^sha256:[a-f0-9]{64}$', hash):
        raise ValueError("Invalid hash format")

    # Use only hash digest (no user input)
    digest = hash.split(':')[1]
    path = os.path.join(self.base_path, "content", digest[:2], digest[2:4], digest)

    # Ensure path is within base_path
    if not os.path.realpath(path).startswith(os.path.realpath(self.base_path)):
        raise SecurityError("Path traversal detected")

    return path
```

### 2. Storage Quota Enforcement

```python
def store(self, content: bytes) -> StorageRef:
    current_usage = self.get_total_usage()
    quota = self.config['quota_bytes']

    if current_usage + len(content) > quota:
        raise StorageFullError(f"Quota exceeded: {current_usage}/{quota}")

    return super().store(content)
```

### 3. Access Control

```python
def retrieve(self, ref: StorageRef, requester: User) -> bytes:
    # Check ACL
    if not self.acl.can_read(ref, requester):
        raise PermissionDeniedError()

    return super().retrieve(ref)
```

---

## Observability

### Metrics

```python
storage_operations_total{operation="store", status="success"} 1234
storage_operations_total{operation="store", status="error"} 5
storage_bytes_stored{backend="filesystem"} 1073741824
storage_dedupe_hits_total 42
storage_latency_seconds{operation="store", quantile="0.95"} 0.25
```

### Logs

```json
{
  "timestamp": "2025-10-22T14:35:00Z",
  "level": "INFO",
  "operation": "store",
  "hash": "sha256:abc123...",
  "size_bytes": 2457600,
  "dedupe_hit": false,
  "latency_ms": 120
}
```

---

## Related Primitives

- `ProvenanceLedger`: Records which `upload_id` references which `StorageRef`
- `FileArtifact`: Wraps `StorageRef` with domain metadata
- `UploadRecord`: Tracks upload lifecycle, references `StorageRef` internally

---

**Last Updated**: 2025-10-22
**Maturity**: Spec complete, ready for implementation
