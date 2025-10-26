# OL Primitive: HashCalculator

**Type**: Utility / Cryptography
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification

---

## Purpose

Streaming hash calculation for large files without loading entire content into memory. Enables deduplication and integrity verification.

---

## Interface Contract

```typescript
interface HashCalculator {
  // Calculate hash from bytes (in-memory)
  calculateHash(content: Bytes): Hash

  // Calculate hash from stream (memory-efficient)
  calculateHashStreaming(stream: ReadableStream): Hash

  // Verify hash matches content
  verifyHash(content: Bytes, expected_hash: Hash): boolean
}

type Hash = string  // Format: "sha256:hexdigest"
```

---

## Implementation

### In-Memory (Small Files)

```python
import hashlib

class HashCalculator:
    def __init__(self, algorithm: str = "sha256"):
        self.algorithm = algorithm

    def calculate_hash(self, content: bytes) -> str:
        hasher = hashlib.sha256()
        hasher.update(content)
        digest = hasher.hexdigest()
        return f"sha256:{digest}"
```

---

### Streaming (Large Files)

```python
def calculate_hash_streaming(self, stream: BinaryIO, chunk_size: int = 8192) -> str:
    """
    Calculate hash without loading entire file into memory.

    Memory usage: O(chunk_size) regardless of file size
    """
    hasher = hashlib.sha256()

    while True:
        chunk = stream.read(chunk_size)
        if not chunk:
            break
        hasher.update(chunk)

    digest = hasher.hexdigest()
    return f"sha256:{digest}"
```

**Example usage:**

```python
with open("large_file.pdf", "rb") as f:
    hash = calculator.calculate_hash_streaming(f)
```

---

### Multipart Upload (HTTP Streaming)

```python
def calculate_hash_from_request(request: HTTPRequest) -> str:
    """
    Calculate hash during upload (before saving to disk).
    """
    hasher = hashlib.sha256()

    for chunk in request.iter_content(chunk_size=8192):
        hasher.update(chunk)

    return f"sha256:{hasher.hexdigest()}"
```

---

## Configuration

```yaml
hash_calculator:
  algorithm: "sha256"  # or "sha512", "blake2b"
  chunk_size_kb: 8
  verify_on_retrieve: true  # Re-hash on retrieval to detect corruption
```

---

## Performance Characteristics

| File Size | In-Memory | Streaming | Notes |
|-----------|-----------|-----------|-------|
| 1MB | 5ms | 10ms | Overhead of chunking |
| 10MB | 50ms | 60ms | In-memory starts thrashing |
| 100MB | OOM risk | 600ms | Streaming wins |
| 1GB | ❌ Crash | 6s | Only streaming viable |

**Recommendation**: Always use streaming for files > 10MB

---

## Algorithm Comparison

| Algorithm | Digest Length | Speed | Collision Resistance |
|-----------|---------------|-------|----------------------|
| SHA-256 | 64 hex chars | Fast | Excellent (2^256) |
| SHA-512 | 128 hex chars | Slower | Excellent (2^512) |
| BLAKE2b | 128 hex chars | Fastest | Excellent |
| MD5 | 32 hex chars | Very fast | ❌ Broken (collisions found) |

**Recommendation**: SHA-256 (standard, well-tested, fast enough)

---

## Testing

```python
def test_consistent_hashing():
    content = b"hello world"

    hash1 = calculator.calculate_hash(content)
    hash2 = calculator.calculate_hash(content)

    assert hash1 == hash2

def test_streaming_matches_in_memory():
    content = b"test data" * 10000  # 90KB

    in_memory_hash = calculator.calculate_hash(content)

    stream = BytesIO(content)
    streaming_hash = calculator.calculate_hash_streaming(stream)

    assert in_memory_hash == streaming_hash

def test_known_hash():
    # Verify against known SHA-256 hash
    content = b"The quick brown fox jumps over the lazy dog"
    expected = "sha256:d7a8fbb307d7809469ca9abcb0082e4f8d5651e46d3cdb762d02d0bf37c9e592"

    assert calculator.calculate_hash(content) == expected
```

---

## Security Considerations

### 1. Collision Resistance

**SHA-256**: No practical collisions known (as of 2025)

**Attack scenario**: Attacker uploads File A (malicious), gets hash H. Later uploads File B (benign) engineered to have same hash H.

**Mitigation**: SHA-256 collision resistance makes this computationally infeasible (2^128 operations).

### 2. Hash as Capability

**Risk**: If hash is exposed publicly, anyone with hash can retrieve content.

**Mitigation**: Don't expose `file_hash` in public API. Use `upload_id` + access control.

---

## Extension Points

### Multi-Algorithm Support

```python
class MultiHashCalculator(HashCalculator):
    def calculate_hash(self, content: bytes) -> Dict[str, str]:
        return {
            "sha256": self._sha256(content),
            "blake2b": self._blake2b(content)
        }
```

**Use case**: Future-proofing (if SHA-256 broken, have backup hash)

---

## Multi-Domain Applicability

This primitive constructs verifiable truth about **content identity through cryptographic hashing** - a universal concept across ALL domains:

**Finance Domain:**
- Dedupe uploaded statements
- Verify transaction document integrity

**Healthcare Domain:**
- Dedupe lab results, medical images
- Verify patient record integrity (HIPAA)

**Legal Domain:**
- Verify document integrity (chain of custody)
- Detect tampering in evidence files

**Research Domain (RSRCH - Utilitario):**
- Verify integrity of scraped web pages (detect if TechCrunch article changed)
- Detect duplicate podcast transcripts (same Lex Fridman episode scraped twice)
- Ensure reproducibility of fact extraction (same source hash → same facts)

**Software Development:**
- Verify build artifacts
- Detect code changes (Git uses SHA-1 hashing)

**Digital Forensics:**
- Evidence integrity verification
- Timeline reconstruction

**Generic Systems:**
- Any file deduplication system
- Any system requiring tamper detection
- Any system needing content-addressable storage

---

**Last Updated**: 2025-10-22
**Maturity**: Spec complete, ready for implementation
