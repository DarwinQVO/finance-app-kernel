# ADR-0043: Three-Tier Scaling Strategy

**Status**: Accepted
**Date**: 2025-10-27
**Context**: Phase 2 (Simplicity Profiles implementation)
**Deciders**: System architects
**Related**: ADR-0042 (Simplicity Profiles Pattern), Phase 3 (Finance App Implementation)

---

## Context

When designing universal primitives, we must decide how many complexity tiers to support. Options range from 2 (Simple/Complex) to 5+ (Micro/Small/Medium/Large/Enterprise).

**Real-world use cases we must support:**
1. **Darwin** (Personal): 500 tx/month, 1 user, SQLite, ~/.finance-app/
2. **Small consulting firm**: 45K tx/month, 10 users, PostgreSQL, single server
3. **FinTech startup**: 8.5M tx/month, 1000+ users, Kubernetes, multi-region

**The Question:**
How many tiers provide meaningful distinctions without creating unnecessary complexity?

---

## Decision

**We use exactly 3 tiers: Personal, Small Business, Enterprise.**

| Tier | Scale | LOC Range | Infrastructure |
|------|-------|-----------|----------------|
| **Personal** | 500 tx/month | 0-100 LOC | Local machine, SQLite, filesystem |
| **Small Business** | 45K tx/month | 50-300 LOC | Single server, PostgreSQL, basic monitoring |
| **Enterprise** | 8.5M+ tx/month | 200-1000 LOC | Distributed, K8s, full observability |

**Each tier represents 10-100× scale jump from previous.**

---

## Rationale

### 1. Three Tiers Match Natural Scaling Boundaries

**Boundary 1: Personal → Small Business** (~100× scale)
- **Trigger**: Single-user → Multi-user
- **Infrastructure change**: Local storage → Database
- **Example**: Darwin adds his wife as user → needs PostgreSQL, user authentication

**Boundary 2: Small Business → Enterprise** (~100× scale)
- **Trigger**: Single-server → Distributed systems
- **Infrastructure change**: Monolith → Microservices
- **Example**: Consulting firm grows to 1000 clients → needs Kubernetes, multi-tenancy, SLAs

**Why these boundaries:**
- Each jump requires architectural redesign (not just "add more servers")
- Clear infrastructure transitions (filesystem → DB → distributed)
- Observable pain points trigger upgrades

### 2. Two Tiers Are Insufficient

**Problem with "Simple/Complex" binary:**

```
Simple (1-100K tx/month): Too wide a range
  - 1 user (Darwin) vs 50 users (small business) have different needs
  - SQLite works for 1K tx, fails at 100K tx
  - No intermediate step between "local filesystem" and "enterprise distributed systems"

Complex (100K+ tx/month): Overkill for small businesses
  - Forces 10-user businesses to adopt Kubernetes
  - Unnecessary operational complexity
  - 10× higher infrastructure costs
```

**Three tiers provide meaningful intermediate step:**
- Personal (1 user, local)
- Small Business (5-50 users, single server) ← Missing in 2-tier model
- Enterprise (100+ users, distributed)

### 3. Four+ Tiers Create Fragmentation

**Problem with 4-tier model (Micro/Small/Medium/Enterprise):**

| Issue | Impact |
|-------|--------|
| **Unclear boundaries** | When is "Small" vs "Medium"? 20 users? 30? 50? |
| **Documentation overhead** | 33 primitives × 4 tiers = 132 profiles to maintain |
| **Decision paralysis** | Users unsure which tier to choose |
| **Minimal differentiation** | "Medium" often just "Small with more RAM" |

**Our 3-tier model has CLEAR boundaries:**
- Personal: 1 user (unambiguous)
- Small Business: Multi-user, single server (clear infrastructure boundary)
- Enterprise: Distributed systems (clear architecture boundary)

---

## Consequences

### Positive

✅ **Clear upgrade triggers**: Each tier has observable pain points that signal need for next tier
✅ **10-100× scale jumps**: Each tier handles order-of-magnitude more load than previous
✅ **Infrastructure alignment**: Tiers match natural infrastructure transitions (local → server → distributed)
✅ **Manageable documentation**: 33 primitives × 3 tiers = 99 profiles (reasonable)
✅ **No "tweener" confusion**: No ambiguous middle tiers

### Negative

⚠️ **Gap at 100K-1M tx/month range**: Small Business might stretch, Enterprise might be overkill
⚠️ **Binary decision at boundaries**: No gradual transition between tiers

### Mitigations

**100K-1M tx/month gap:**
- Small Business profile designed to stretch to 100K tx/month with vertical scaling
- Enterprise profile has "Small Enterprise" configuration (PostgreSQL before Kubernetes)
- Pain points documented: "If PostgreSQL hitting limits → add read replicas → eventually K8s"

**Binary transitions:**
- Upgrade paths are incremental within tiers first
- Example: Personal → Personal + PostgreSQL → Small Business
- No "big bang" migration required

---

## Decision Criteria Per Tier

### Personal Profile

**Use when:**
- Single user (or 2-3 close collaborators)
- < 1K transactions/month
- Local machine deployment
- Uptime requirements: "whenever my laptop is on"
- Budget: $0 (no infrastructure costs)

**Infrastructure:**
- SQLite (built-in, no server)
- Local filesystem (~/.finance-app/)
- No authentication (trust local user)
- No monitoring (user IS the monitor)

**Upgrade triggers:**
- Need multi-user access from different devices → Small Business
- Database file > 1GB → Small Business
- Need backup/recovery → Small Business

### Small Business Profile

**Use when:**
- 5-50 users
- 1K-100K transactions/month
- Single server (can be large, but single instance)
- Uptime requirements: 99% (8.7 hours downtime/year acceptable)
- Budget: $100-500/month (single VPS)

**Infrastructure:**
- PostgreSQL (standard RDBMS)
- Single application server
- Basic auth (username/password, maybe OAuth)
- Simple monitoring (Prometheus + Grafana)
- Daily backups

**Upgrade triggers:**
- Server hitting CPU/memory limits → Enterprise
- Need multi-region deployment → Enterprise
- Compliance requirements (SOC 2, HIPAA) → Enterprise
- Need 99.9%+ uptime → Enterprise

### Enterprise Profile

**Use when:**
- 100+ users (or 10K+ end customers)
- 100K+ transactions/month
- Distributed systems required
- Uptime requirements: 99.9%+ (< 9 hours downtime/year)
- Budget: $1K+/month (multi-server, managed services)

**Infrastructure:**
- PostgreSQL cluster (or managed DB like RDS)
- Kubernetes (container orchestration)
- Full auth suite (OAuth2, OIDC, MFA)
- Complete observability (metrics, logs, traces)
- Hourly backups, point-in-time recovery

**Characteristics:**
- No upgrade path from Enterprise (this is the ceiling)
- Further scaling is horizontal (more servers, not different architecture)

---

## Real-World Mapping

### Darwin (Personal)

**Stats:**
- Users: 1 (Darwin)
- Transactions: 500/month (42 per statement, 12 statements/year)
- Storage: ~100MB total (PDFs + SQLite)
- Deployment: MacBook Pro, ~/.finance-app/

**Profile fit:** Personal (uses 3.8% of Enterprise code: 1,365 LOC / 35,700 LOC)

**Upgrade trigger:** If Darwin's wife starts using the app from her laptop → Small Business

---

### Small Consulting Firm

**Stats:**
- Users: 15 (consultants)
- Transactions: 45K/month (3K per user)
- Storage: ~5GB total
- Deployment: DigitalOcean droplet ($50/month)

**Profile fit:** Small Business (~4,200 LOC total)

**Upgrade trigger:** If firm grows to 50+ consultants or lands Fortune 500 client (compliance) → Enterprise

---

### FinTech Startup

**Stats:**
- Users: 2,000 (employees + partners)
- Transactions: 8.5M/month
- Storage: ~2TB total
- Deployment: AWS (EKS, RDS, S3)

**Profile fit:** Enterprise (~35,700 LOC total)

**No further tiers:** Scaling is horizontal (more Kubernetes pods, bigger RDS instance)

---

## Alternatives Considered

### Alternative 1: Two Tiers (Simple/Complex)

**Approach:** Binary choice.

| Tier | Scale |
|------|-------|
| Simple | 1-100K tx/month |
| Complex | 100K+ tx/month |

**Rejected because:**
- Too large a gap in "Simple" (1 user vs 50 users have very different needs)
- No intermediate step between "local filesystem" and "Kubernetes"
- Small businesses forced to choose between underpowered or overpowered

---

### Alternative 2: Four Tiers (Micro/Small/Medium/Enterprise)

**Approach:** Finer granularity.

| Tier | Scale |
|------|-------|
| Micro | 1-1K tx/month |
| Small | 1K-10K tx/month |
| Medium | 10K-100K tx/month |
| Enterprise | 100K+ tx/month |

**Rejected because:**
- Unclear boundaries (Small vs Medium often arbitrary)
- Documentation overhead (33 × 4 = 132 profiles)
- "Medium" often just "Small with more RAM" (not architectural change)

---

### Alternative 3: Five Tiers (Personal/Startup/SMB/Mid-Market/Enterprise)

**Approach:** Match "market segments."

**Rejected because:**
- Market segments don't align with technical architecture
- Creates "tweener" tiers with ambiguous characteristics
- No clear infrastructure transitions

---

## Implementation Notes

### Tier Detection (Auto-recommendation)

Each primitive can suggest appropriate tier based on usage metrics:

```python
def recommend_profile(stats: UsageStats) -> Profile:
    if stats.users <= 3 and stats.tx_per_month < 1000:
        return Profile.PERSONAL
    elif stats.users < 50 and stats.tx_per_month < 100000:
        return Profile.SMALL_BUSINESS
    else:
        return Profile.ENTERPRISE
```

### Gradual Upgrades Within Tiers

**Personal → Personal+:**
- Start: SQLite
- Add: PostgreSQL (keep local deployment)
- Still "Personal" until multi-user authentication needed

**Small Business → Small Business+:**
- Start: Single PostgreSQL server
- Add: Read replicas (horizontal scaling within tier)
- Still "Small Business" until Kubernetes needed

---

## Related Decisions

- **ADR-0042:** Simplicity Profiles Pattern (defines profile structure)
- **ADR-0044:** YAGNI Principle in Personal Profile (justifies 0 LOC features)
- **ADR-0045:** Hardcoded Configuration Strategy (Personal tier uses hardcoding extensively)

---

## Review & Updates

- **2025-10-27:** Initial decision (based on Phase 2 implementation experience)
- Next review: After 20+ real-world deployments across all tiers

---

**Decision**: Use exactly 3 tiers (Personal, Small Business, Enterprise) with 10-100× scale jumps.
**Status**: Accepted and implemented across all 33 primitives.
