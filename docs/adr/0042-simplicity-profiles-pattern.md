# ADR-0042: Simplicity Profiles Pattern

**Status**: Accepted
**Date**: 2025-10-27
**Context**: Phase 2 (Simplicity Profiles for 33 primitives)
**Deciders**: System architects
**Related**: Phase 1 (Storytelling), Phase 3 (Finance App Implementation)

---

## Context

The kernel defines 33 universal primitives (OL) designed for enterprise scale (10M+ transactions/month, multi-tenant, compliance requirements). However, these same primitives must also work for:

1. **Personal use** (Darwin: 500 tx/month, 1 user, SQLite, local filesystem)
2. **Small business** (45K tx/month, 5-50 users, PostgreSQL, simple multi-tenancy)
3. **Enterprise** (8.5M+ tx/month, 1000+ users, distributed systems, compliance)

**The Problem:**
- Writing 3 separate implementations violates DRY
- Single implementation forces all users to pay enterprise costs
- Traditional "configuration flags" become unmaintainable (100s of if/else branches)

**The Question:**
How do we make the same primitives work at radically different scales without code duplication or complexity explosion?

---

## Decision

**We use Simplicity Profiles: Same interface, radically different implementations at 3 complexity tiers.**

Each primitive documents:
- **Personal Profile** (0-100 LOC): Minimal viable implementation, hardcoded assumptions, YAGNI applied aggressively
- **Small Business Profile** (50-300 LOC): Configurable rules, basic multi-tenancy, standard patterns
- **Enterprise Profile** (200-1000 LOC): Full feature set, ML, distributed systems, compliance, observability

**Key Principle:** Interface consistency, implementation diversity.

---

## Rationale

### 1. Interface Consistency

**Same interface works at all scales:**

```python
# Personal (5 LOC)
class HashCalculator:
    def calculate_hash(self, data: bytes) -> str:
        return f"sha256:{hashlib.sha256(data).hexdigest()}"

# Enterprise (200 LOC)
class HashCalculator:
    def calculate_hash(self, data: bytes) -> str:
        # Same signature, different implementation:
        # - Multi-algorithm support (SHA-256, SHA-512, BLAKE3)
        # - Streaming for large files
        # - Virus scanning integration
        # - Malware hash checking
        # - Metrics collection
        return f"{self.algorithm}:{hash_value}"
```

**Benefits:**
- Darwin can upgrade from Personal → Small Business → Enterprise without code changes
- Tests written for Personal work for Enterprise (contract compliance)
- Documentation applies to all scales

### 2. YAGNI Applied Per Scale

**Personal Profile omits enterprise features:**

Example: `DuplicateDetector`
- **Personal**: 0 LOC - Not needed (user manually marks duplicates, happens <5x/year)
- **Small Business**: 150 LOC - Fuzzy matching (string similarity, date proximity)
- **Enterprise**: 1500 LOC - ML clustering, confidence scores, auto-resolution, audit trail

**Rationale:**
- Don't force Personal users to pay for features they'll never use
- 0 LOC documents explicit "this isn't needed yet" decision
- Upgrade path clear when pain point emerges

### 3. Optimal Implementations Per Scale

**Each profile uses scale-appropriate patterns:**

Example: `ParserRegistry`
- **Personal**: Dict in memory (1 parser: BoFA PDF)
  ```python
  PARSERS = {"bofa_pdf": BofaParser()}
  ```
- **Small Business**: SQLite table (10-20 parsers)
  ```sql
  CREATE TABLE parsers (parser_id TEXT PRIMARY KEY, ...);
  ```
- **Enterprise**: PostgreSQL + Consul service discovery (100+ parsers, versioning, canary deploys)
  ```python
  registry = ConsulParserRegistry(consul_url="...")
  ```

**Benefits:**
- Personal: 5 LOC, zero dependencies
- Small Business: 100 LOC, SQLite included
- Enterprise: 800 LOC, production-grade infrastructure

### 4. Literate Programming Style

**Profiles tell a story:**

Each profile explains:
- **What's included** (features enabled)
- **What's excluded** (features omitted, why)
- **When to upgrade** (pain points that trigger next tier)
- **Configuration** (YAML example)
- **Performance characteristics** (latency, memory, throughput)

**Example narrative:**
> "Darwin has 1 bank account (BoFA) and uploads statements monthly. A hardcoded dict is appropriate because the account list is stable. If Darwin adds 10+ accounts, upgrade to Small Business profile (database storage enables UI for account management)."

---

## Consequences

### Positive

✅ **Same interfaces, all scales**: Write once, run anywhere (with appropriate profile)
✅ **No feature bloat**: Personal users don't pay for enterprise complexity
✅ **Clear upgrade path**: Pain point triggers documented in each profile
✅ **Optimal performance**: Each profile uses scale-appropriate patterns (dict vs SQLite vs PostgreSQL)
✅ **Maintainability**: Single source of truth (interface), 3 well-documented implementations
✅ **Testability**: Interface contract ensures all profiles interchangeable

### Negative

⚠️ **Implementation duplication**: 3 implementations per primitive (mitigated by shared interface)
⚠️ **Documentation overhead**: Must document all 3 profiles per primitive (33 × 3 = 99 profiles)
⚠️ **Decision fatigue**: Users must choose profile (mitigated by clear decision criteria)

### Mitigations

**Implementation duplication:**
- Acceptable because implementations are RADICALLY different (dict vs DB vs distributed system)
- Shared interface ensures behavioral consistency
- Code reuse where appropriate (Personal can import Enterprise utilities)

**Documentation overhead:**
- Upfront cost, ongoing maintenance benefit
- Profiles prevent "unknown complexity" syndrome
- Clear LOC estimates help planning

**Decision fatigue:**
- Decision matrix per primitive: "If X, use Personal; If Y, use Small Business; If Z, use Enterprise"
- Default to Personal, upgrade on pain points

---

## Alternatives Considered

### Alternative 1: Single Implementation with Feature Flags

**Approach:** One implementation, control complexity via configuration.

```python
class HashCalculator:
    def __init__(self, config):
        self.enable_streaming = config.get("streaming", False)
        self.enable_virus_scan = config.get("virus_scan", False)
        self.enable_metrics = config.get("metrics", False)
        # ... 50 more flags

    def calculate_hash(self, data):
        if self.enable_streaming:
            # Streaming logic
        else:
            # In-memory logic
        if self.enable_virus_scan:
            # Virus scanning
        # ... 100 more if/else branches
```

**Rejected because:**
- Complexity explosion (every primitive becomes 500+ LOC with nested ifs)
- Personal users pay maintenance cost for enterprise features they never use
- Testing nightmare (2^50 configuration combinations)
- No clear "this is Personal" vs "this is Enterprise" boundary

---

### Alternative 2: Separate Codebases Per Scale

**Approach:** 3 independent implementations.

```
kernel-personal/
kernel-small-business/
kernel-enterprise/
```

**Rejected because:**
- Interface drift (3 implementations diverge over time)
- Bug fixes must be applied 3× (high maintenance cost)
- No upgrade path (Personal → Enterprise requires rewrite)
- Documentation fragmentation

---

### Alternative 3: Inheritance Hierarchy

**Approach:** `PersonalHashCalculator` extends `BaseHashCalculator`, `EnterpriseHashCalculator` extends `PersonalHashCalculator`.

```python
class BaseHashCalculator:
    def calculate_hash(self, data): ...

class PersonalHashCalculator(BaseHashCalculator):
    def calculate_hash(self, data):
        return super().calculate_hash(data)  # Basic impl

class EnterpriseHashCalculator(PersonalHashCalculator):
    def calculate_hash(self, data):
        # Add virus scanning, metrics, etc.
        return super().calculate_hash(data)
```

**Rejected because:**
- Forces inheritance coupling (Enterprise inherits Personal complexity)
- "Everything but the kitchen sink" problem (Personal gets enterprise imports)
- Fragile (changes to Personal break Enterprise)
- Violates "composition over inheritance"

**Current approach:** Composition with shared interfaces, independent implementations.

---

## Implementation Notes

### Profile Structure Per Primitive

Each primitive file includes 3 sections:

```markdown
## Simplicity Profiles

### Personal Profile (0-100 LOC)
- **When to use**: < 1K transactions/month, 1 user, local machine
- **Implementation**: [Minimal code example]
- **Configuration**: [YAML config]
- **What's included**: [Feature list]
- **What's excluded**: [Omitted features, rationale]
- **Performance**: [Latency, memory]

### Small Business Profile (50-300 LOC)
- **When to use**: 1K-100K transactions/month, 5-50 users, single server
- **Implementation**: [Configurable code example]
- **Configuration**: [YAML config]
- **Upgrade triggers**: [Pain points from Personal]
- **Performance**: [Latency, memory]

### Enterprise Profile (200-1000 LOC)
- **When to use**: 100K+ transactions/month, 100+ users, distributed systems
- **Implementation**: [Full-featured code example]
- **Configuration**: [YAML config]
- **Upgrade triggers**: [Pain points from Small Business]
- **Performance**: [Latency, memory, scalability]
```

### Decision Matrix

Each primitive includes decision criteria:

```yaml
profile_selection:
  personal:
    - transaction_count < 1000/month
    - users == 1
    - uptime_requirement < 99%

  small_business:
    - transaction_count < 100000/month
    - users < 50
    - uptime_requirement >= 99%

  enterprise:
    - transaction_count >= 100000/month
    - users >= 100
    - compliance_required == true
    - uptime_requirement >= 99.9%
```

---

## Related Decisions

- **ADR-0043:** Three-Tier Scaling Strategy (Personal → Small Business → Enterprise tiers)
- **ADR-0044:** YAGNI Principle in Personal Profile (0 LOC = explicit "not needed" decision)
- **ADR-0045:** Hardcoded Configuration Strategy (when dicts beat databases)
- **Phase 3:** Finance App Implementation (shows Darwin using Personal profiles: 1,365 LOC total)

---

## Review & Updates

- **2025-10-27:** Initial decision (Phase 2 completion: 33/33 primitives profiled)
- Next review: After 10+ real-world implementations, assess pattern effectiveness

---

**Decision**: Use Simplicity Profiles (same interface, 3 implementations per scale tier).
**Status**: Accepted and implemented across all 33 universal primitives.
