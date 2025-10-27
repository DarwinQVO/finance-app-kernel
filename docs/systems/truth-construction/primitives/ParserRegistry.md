# OL Primitive: ParserRegistry

**Type**: Service Discovery / Registry
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.2 (Extraction)

---

## Purpose

Central registry for discovering and selecting appropriate Parser based on `source_type`. Enables pluggable parser architecture - add new parsers without modifying core extraction logic.

---

## Interface Contract

```python
from typing import Optional, List

class ParserRegistry:
    def register(self, parser: Parser) -> None:
        """Register a parser. Raises error if parser_id already registered."""
        pass

    def get_parser(self, source_type: str) -> Parser:
        """
        Get parser for source_type.
        Raises: ParserNotFoundError if no parser supports source_type
        """
        pass

    def get_all_parsers(self) -> List[Parser]:
        """Return all registered parsers."""
        pass

    def supports(self, source_type: str) -> bool:
        """Check if any parser supports source_type."""
        pass
```

---

## Behavior

**Registration:**
```python
registry = ParserRegistry()
registry.register(BofAPDFParser())
registry.register(ChaseCSVParser())
```

**Discovery:**
```python
parser = registry.get_parser("bofa_pdf")  # Returns BofAPDFParser
parser = registry.get_parser("chase_csv")  # Returns ChaseCSVParser
parser = registry.get_parser("unknown")    # Raises ParserNotFoundError
```

---

## Multi-Domain Applicability

**Finance:** Register bank-specific parsers (BoFA, Chase, Wells Fargo)
**Healthcare:** Register lab-specific parsers (LabCorp, Quest, hospital systems)
**Legal:** Register contract parsers (DocuSign, HelloSign, manual PDF)
**Research (RSRCH - Utilitario):** Register web scraping parsers (TechCrunch parser, Twitter parser, Lex Fridman podcast transcript parser)
**Manufacturing:** Register sensor parsers (CSV, JSON, proprietary formats)

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Register bank-specific parsers for statement extraction
**Example:** System startup registers parsers → registry.register(BofAPDFParser()), registry.register(ChaseCSVParser()), registry.register(AmexOFXParser()) → User uploads "bofa_statement.pdf" with source_type="bofa_pdf" → Coordinator calls registry.get_parser("bofa_pdf") → Returns BofAPDFParser instance → Parser extracts 42 transactions
**Operations:** register (add parser), get_parser (lookup by source_type), supports (check availability), get_all_parsers (list all)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Register lab-specific parsers for test result extraction
**Example:** Registry registers LabCorpPDFParser, QuestPDFParser → Patient uploads "quest_labs.pdf" with source_type="quest_pdf" → registry.get_parser("quest_pdf") → Returns QuestPDFParser → Extracts 8 lab test observations
**Status:** ✅ Conceptually validated

### ✅ Legal
**Use case:** Register contract parsers for document processing
**Example:** Registry registers DocuSignPDFParser, HelloSignPDFParser → Law firm uploads "contract.pdf" with source_type="docusign_pdf" → registry.get_parser("docusign_pdf") → Returns parser → Extracts 25 clauses
**Status:** ✅ Conceptually validated

### ✅ RSRCH (Utilitario Research)
**Use case:** Register web scraping parsers for founder fact extraction
**Example:** Registry registers TechCrunchHTMLParser, TwitterJSONParser, LexFridmanTranscriptParser → Scraper downloads TechCrunch article with source_type="techcrunch_html" → registry.get_parser("techcrunch_html") → Returns TechCrunchHTMLParser → Extracts 12 founder investment facts (RawFact records: "@sama invested $375M in OpenAI")
**Operations:** Pluggable parser architecture critical for multi-source research (web, podcasts, tweets), registry.supports("source_type") checks availability before scraping
**Status:** ✅ Conceptually validated

### ✅ E-commerce
**Use case:** Register supplier catalog parsers for product extraction
**Example:** Registry registers SupplierCSVParser, AmazonAPIParser → Merchant imports "supplier_catalog.csv" with source_type="supplier_csv" → registry.get_parser("supplier_csv") → Returns parser → Extracts 1,500 product observations
**Status:** ✅ Conceptually validated

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (service discovery pattern, no domain-specific code)
**Reusability:** High (same register/get_parser operations work for bank parsers, lab parsers, contract parsers, web scrapers, catalog parsers)

---

## Simplicity Profiles

The same ParserRegistry interface supports three implementation scales: Personal (hardcoded dictionary), Small Business (YAML config), Enterprise (database with versioning + canary deployments).

### Personal Profile (30 LOC)

**Darwin's Approach:** Hardcoded dictionary mapping `source_type` to parser instances

**What Darwin needs:**
- Map "bofa_pdf" → BofAPDFParser
- Raise error if parser not found
- Simple, zero configuration

**What Darwin DOESN'T need:**
- ❌ YAML config file (Darwin only has 1 bank account)
- ❌ Dynamic parser registration
- ❌ Parser versioning
- ❌ Canary deployments (gradual rollout of new parsers)
- ❌ A/B testing (compare parser performance)

**Implementation:**

**Code Example (30 LOC):**
```python
# parser_registry.py (Darwin's personal finance app)

from parsers.bofa_pdf import BofAPDFParser

class ParserNotFoundError(Exception):
    pass

class ParserRegistry:
    """Simple hardcoded parser registry."""

    def __init__(self):
        # Hardcoded: Darwin only uses Bank of America
        self._parsers = {
            "bofa_pdf": BofAPDFParser()
        }

    def get_parser(self, source_type: str):
        """Get parser for source_type."""
        if source_type not in self._parsers:
            raise ParserNotFoundError(
                f"No parser found for source_type: {source_type}. "
                f"Available: {list(self._parsers.keys())}"
            )
        return self._parsers[source_type]

    def supports(self, source_type: str) -> bool:
        """Check if parser exists for source_type."""
        return source_type in self._parsers
```

**Usage:**
```python
# In extraction coordinator
registry = ParserRegistry()

# Get parser for uploaded PDF
parser = registry.get_parser("bofa_pdf")
observations = parser.parse(pdf_path)

# Check support before parsing
if registry.supports("chase_csv"):
    parser = registry.get_parser("chase_csv")
else:
    raise ValueError("Chase CSV not supported yet")
```

**Why hardcoded is correct:**
- Darwin only has 1 bank account (Bank of America)
- Parser selection will never change at runtime
- No need for configuration file
- If Darwin switches banks in the future → 2-line code change

**Example Error:**
```python
>>> registry.get_parser("chase_csv")
ParserNotFoundError: No parser found for source_type: chase_csv.
Available: ['bofa_pdf']
```

**Testing:**
- ❌ No tests (trivial dictionary lookup, can't fail)
- Manual verification: Upload PDF, check parser executes

**Decision Context:**
- Darwin has 1 bank → 1 parser → hardcoded dictionary is sufficient
- YAGNI: No need for dynamic registration when you have 1 parser
- If Darwin adds Chase account later → Add one line to `_parsers` dict

---

### Small Business Profile (120 LOC)

**Use Case:** Accounting firm processing statements from 8 different banks

**What they need (vs Personal):**
- ✅ All Personal features (get_parser, supports, error handling)
- ✅ **YAML config file** (easier to add new banks without code changes)
- ✅ **Dynamic registration** (load parsers from config at startup)
- ✅ **Parser metadata** (description, supported formats)
- ❌ No versioning (single version per parser)
- ❌ No canary deployments (too complex for 50 statements/month)
- ❌ No A/B testing (not enough volume)

**Implementation:**

**Config File (config/parsers.yaml):**
```yaml
parsers:
  - source_type: "bofa_pdf"
    class_name: "parsers.bofa_pdf.BofAPDFParser"
    description: "Bank of America PDF statement parser"
    supported_formats: ["pdf"]

  - source_type: "chase_csv"
    class_name: "parsers.chase_csv.ChaseCSVParser"
    description: "Chase Bank CSV export parser"
    supported_formats: ["csv"]

  - source_type: "wells_fargo_pdf"
    class_name: "parsers.wells_fargo_pdf.WellsFargoPDFParser"
    description: "Wells Fargo PDF statement parser"
    supported_formats: ["pdf"]

  - source_type: "amex_ofx"
    class_name: "parsers.amex_ofx.AmexOFXParser"
    description: "American Express OFX file parser"
    supported_formats: ["ofx", "qfx"]

  - source_type: "citi_pdf"
    class_name: "parsers.citi_pdf.CitiPDFParser"
    description: "Citibank PDF statement parser"
    supported_formats: ["pdf"]
```

**Code Example (120 LOC):**
```python
# parser_registry.py (Small Business)

import yaml
import importlib
from pathlib import Path
from typing import Dict, List, Optional

class ParserNotFoundError(Exception):
    pass

class ParserMetadata:
    """Metadata about a registered parser."""
    def __init__(self, source_type: str, class_name: str, description: str, supported_formats: List[str]):
        self.source_type = source_type
        self.class_name = class_name
        self.description = description
        self.supported_formats = supported_formats

class ParserRegistry:
    """YAML-configured parser registry with dynamic loading."""

    def __init__(self, config_path: str = "config/parsers.yaml"):
        self._parsers: Dict[str, object] = {}
        self._metadata: Dict[str, ParserMetadata] = {}
        self._load_from_config(config_path)

    def _load_from_config(self, config_path: str):
        """Load parsers from YAML config."""
        config = yaml.safe_load(Path(config_path).read_text())

        for parser_def in config.get("parsers", []):
            source_type = parser_def["source_type"]
            class_name = parser_def["class_name"]
            description = parser_def.get("description", "")
            supported_formats = parser_def.get("supported_formats", [])

            # Store metadata
            self._metadata[source_type] = ParserMetadata(
                source_type=source_type,
                class_name=class_name,
                description=description,
                supported_formats=supported_formats
            )

            # Dynamically import and instantiate parser
            module_path, class_name_only = class_name.rsplit(".", 1)
            module = importlib.import_module(module_path)
            parser_class = getattr(module, class_name_only)
            self._parsers[source_type] = parser_class()

    def get_parser(self, source_type: str):
        """Get parser for source_type."""
        if source_type not in self._parsers:
            available = ", ".join(self._parsers.keys())
            raise ParserNotFoundError(
                f"No parser found for source_type: {source_type}. "
                f"Available: {available}"
            )
        return self._parsers[source_type]

    def supports(self, source_type: str) -> bool:
        """Check if parser exists for source_type."""
        return source_type in self._parsers

    def get_all_parsers(self) -> List[str]:
        """Return list of all registered parser source_types."""
        return list(self._parsers.keys())

    def get_metadata(self, source_type: str) -> Optional[ParserMetadata]:
        """Get metadata for a parser."""
        return self._metadata.get(source_type)

    def list_parsers(self) -> List[Dict]:
        """List all parsers with metadata (for admin UI)."""
        result = []
        for source_type, metadata in self._metadata.items():
            result.append({
                "source_type": source_type,
                "description": metadata.description,
                "supported_formats": metadata.supported_formats,
                "class_name": metadata.class_name
            })
        return result
```

**Usage:**
```python
# Load registry from config
registry = ParserRegistry("config/parsers.yaml")

# Get parser
parser = registry.get_parser("chase_csv")

# List all available parsers (for UI dropdown)
for info in registry.list_parsers():
    print(f"{info['source_type']}: {info['description']}")
```

**Output:**
```
bofa_pdf: Bank of America PDF statement parser
chase_csv: Chase Bank CSV export parser
wells_fargo_pdf: Wells Fargo PDF statement parser
amex_ofx: American Express OFX file parser
citi_pdf: Citibank PDF statement parser
```

**Adding a new bank (no code changes):**
```yaml
# Just add to config/parsers.yaml
parsers:
  # ... existing parsers ...

  - source_type: "usbank_pdf"
    class_name: "parsers.usbank_pdf.USBankPDFParser"
    description: "US Bank PDF statement parser"
    supported_formats: ["pdf"]
```

**Testing:**
- ✅ Unit test: Load config, verify all parsers instantiated
- ✅ Integration test: Mock YAML file, verify dynamic import works
- ✅ Error test: Invalid class_name → raises ImportError with helpful message

**Monitoring:**
- Startup log: "Loaded 8 parsers from config/parsers.yaml"
- Error log if parser fails to import

**Decision Context:**
- Accounting firm has 20 clients across 8 banks
- Adding new banks happens monthly
- YAML config easier than code changes (operations team can edit)
- Version control for config file (git tracks parser additions)

---

### Enterprise Profile (800 LOC)

**Use Case:** Multi-tenant financial platform supporting 50+ banks with gradual parser rollouts

**What they need (vs Small Business):**
- ✅ All Small Business features (YAML config, dynamic loading, metadata)
- ✅ **Database-backed registry** (PostgreSQL with parser versions table)
- ✅ **Parser versioning** (v1.0, v1.1, v2.0 for same bank)
- ✅ **Canary deployments** (gradual rollout: 5% → 25% → 50% → 100%)
- ✅ **A/B testing** (compare performance of two parser versions)
- ✅ **Feature flags** (enable/disable parsers per tenant)
- ✅ **Parser health checks** (success rate, latency monitoring)
- ✅ **Rollback capability** (instantly switch back to previous version)

**Implementation:**

**Database Schema:**
```sql
CREATE TABLE parser_registry (
    id SERIAL PRIMARY KEY,
    source_type TEXT NOT NULL,
    version TEXT NOT NULL,
    class_name TEXT NOT NULL,
    description TEXT,
    supported_formats TEXT[],
    enabled BOOLEAN DEFAULT TRUE,
    rollout_percentage INTEGER DEFAULT 0,  -- 0-100 (canary deployment)
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(source_type, version)
);

CREATE TABLE parser_assignments (
    tenant_id TEXT NOT NULL,
    source_type TEXT NOT NULL,
    parser_version TEXT NOT NULL,
    assignment_type TEXT NOT NULL,  -- 'canary', 'stable', 'override'
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (tenant_id, source_type)
);

CREATE TABLE parser_health (
    source_type TEXT NOT NULL,
    version TEXT NOT NULL,
    success_count INTEGER DEFAULT 0,
    failure_count INTEGER DEFAULT 0,
    avg_duration_ms INTEGER,
    last_health_check TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (source_type, version)
);
```

**Code Example (800 LOC - excerpt):**
```python
# parser_registry.py (Enterprise)

import importlib
import random
from typing import Dict, List, Optional
import psycopg2
from psycopg2.extras import DictCursor

class ParserNotFoundError(Exception):
    pass

class ParserRegistry:
    """Enterprise parser registry with versioning, canary deployments, health checks."""

    def __init__(self, db_conn_string: str):
        self.db_conn_string = db_conn_string
        self._parser_cache: Dict[str, object] = {}  # Cache instantiated parsers
        self._init_db()

    def _init_db(self):
        """Create tables if not exist."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()

        # Schema creation from above
        # ... (omitted for brevity)

        conn.commit()
        conn.close()

    def register_parser(
        self,
        source_type: str,
        version: str,
        class_name: str,
        description: str = "",
        supported_formats: List[str] = None,
        rollout_percentage: int = 0
    ):
        """Register a new parser version."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO parser_registry
            (source_type, version, class_name, description, supported_formats, rollout_percentage)
            VALUES (%s, %s, %s, %s, %s, %s)
            ON CONFLICT (source_type, version) DO UPDATE SET
                class_name = EXCLUDED.class_name,
                description = EXCLUDED.description,
                supported_formats = EXCLUDED.supported_formats,
                rollout_percentage = EXCLUDED.rollout_percentage
        """, (source_type, version, class_name, description, supported_formats or [], rollout_percentage))

        conn.commit()
        conn.close()

        # Clear cache to force reload
        cache_key = f"{source_type}:{version}"
        if cache_key in self._parser_cache:
            del self._parser_cache[cache_key]

    def get_parser(self, source_type: str, tenant_id: Optional[str] = None):
        """
        Get parser for source_type, respecting canary rollout and tenant assignments.

        Logic:
        1. Check if tenant has explicit assignment (override)
        2. Check canary rollout percentage
        3. Fall back to stable version
        """
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor(cursor_factory=DictCursor)

        # 1. Check tenant assignment (highest priority)
        if tenant_id:
            cursor.execute("""
                SELECT parser_version FROM parser_assignments
                WHERE tenant_id = %s AND source_type = %s
            """, (tenant_id, source_type))
            row = cursor.fetchone()
            if row:
                version = row["parser_version"]
                conn.close()
                return self._load_parser(source_type, version)

        # 2. Check canary rollout
        cursor.execute("""
            SELECT version, rollout_percentage FROM parser_registry
            WHERE source_type = %s AND enabled = TRUE
            ORDER BY version DESC
        """, (source_type,))
        rows = cursor.fetchall()
        conn.close()

        if not rows:
            raise ParserNotFoundError(f"No parser found for source_type: {source_type}")

        # Find canary version
        canary_version = None
        stable_version = None

        for row in rows:
            if row["rollout_percentage"] > 0 and row["rollout_percentage"] < 100:
                canary_version = row
            elif row["rollout_percentage"] == 100:
                stable_version = row

        # Randomly assign to canary based on rollout percentage
        if canary_version and random.randint(1, 100) <= canary_version["rollout_percentage"]:
            version = canary_version["version"]
        elif stable_version:
            version = stable_version["version"]
        else:
            # No stable version, use latest
            version = rows[0]["version"]

        return self._load_parser(source_type, version)

    def _load_parser(self, source_type: str, version: str):
        """Load and cache parser instance."""
        cache_key = f"{source_type}:{version}"

        if cache_key in self._parser_cache:
            return self._parser_cache[cache_key]

        # Fetch class_name from DB
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor(cursor_factory=DictCursor)
        cursor.execute("""
            SELECT class_name FROM parser_registry
            WHERE source_type = %s AND version = %s
        """, (source_type, version))
        row = cursor.fetchone()
        conn.close()

        if not row:
            raise ParserNotFoundError(f"Parser {source_type}:{version} not found")

        # Dynamically import and instantiate
        class_name = row["class_name"]
        module_path, class_name_only = class_name.rsplit(".", 1)
        module = importlib.import_module(module_path)
        parser_class = getattr(module, class_name_only)
        parser_instance = parser_class()

        # Cache
        self._parser_cache[cache_key] = parser_instance
        return parser_instance

    def set_canary_rollout(self, source_type: str, version: str, percentage: int):
        """
        Update canary rollout percentage.

        Typical progression:
        - v2.0 at 5% (canary test with small traffic)
        - v2.0 at 25% (expand canary)
        - v2.0 at 50% (half traffic)
        - v2.0 at 100% (promote to stable)
        """
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()
        cursor.execute("""
            UPDATE parser_registry
            SET rollout_percentage = %s
            WHERE source_type = %s AND version = %s
        """, (percentage, source_type, version))
        conn.commit()
        conn.close()

    def assign_tenant_override(self, tenant_id: str, source_type: str, version: str):
        """Assign specific parser version to tenant (for A/B testing or special requirements)."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()
        cursor.execute("""
            INSERT INTO parser_assignments (tenant_id, source_type, parser_version, assignment_type)
            VALUES (%s, %s, %s, 'override')
            ON CONFLICT (tenant_id, source_type) DO UPDATE SET
                parser_version = EXCLUDED.parser_version
        """, (tenant_id, source_type, version))
        conn.commit()
        conn.close()

    def record_health(self, source_type: str, version: str, success: bool, duration_ms: int):
        """Record parser execution health metrics."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()

        if success:
            cursor.execute("""
                INSERT INTO parser_health (source_type, version, success_count, avg_duration_ms)
                VALUES (%s, %s, 1, %s)
                ON CONFLICT (source_type, version) DO UPDATE SET
                    success_count = parser_health.success_count + 1,
                    avg_duration_ms = ((parser_health.avg_duration_ms * parser_health.success_count) + %s) / (parser_health.success_count + 1),
                    last_health_check = NOW()
            """, (source_type, version, duration_ms, duration_ms))
        else:
            cursor.execute("""
                INSERT INTO parser_health (source_type, version, failure_count)
                VALUES (%s, %s, 1)
                ON CONFLICT (source_type, version) DO UPDATE SET
                    failure_count = parser_health.failure_count + 1,
                    last_health_check = NOW()
            """, (source_type, version))

        conn.commit()
        conn.close()

    def get_health_metrics(self, source_type: str) -> List[Dict]:
        """Get health metrics for all versions of a parser."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor(cursor_factory=DictCursor)
        cursor.execute("""
            SELECT version, success_count, failure_count, avg_duration_ms,
                   ROUND(100.0 * success_count / NULLIF(success_count + failure_count, 0), 2) as success_rate
            FROM parser_health
            WHERE source_type = %s
            ORDER BY version DESC
        """, (source_type,))
        rows = cursor.fetchall()
        conn.close()

        return [dict(row) for row in rows]

    def rollback(self, source_type: str, to_version: str):
        """
        Emergency rollback: Instantly switch all traffic to a previous version.

        Sets to_version rollout to 100%, sets all other versions to 0%.
        """
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()

        # Set all versions to 0%
        cursor.execute("""
            UPDATE parser_registry
            SET rollout_percentage = 0
            WHERE source_type = %s
        """, (source_type,))

        # Set target version to 100%
        cursor.execute("""
            UPDATE parser_registry
            SET rollout_percentage = 100
            WHERE source_type = %s AND version = %s
        """, (source_type, to_version))

        conn.commit()
        conn.close()

        # Clear cache
        for key in list(self._parser_cache.keys()):
            if key.startswith(f"{source_type}:"):
                del self._parser_cache[key]
```

**Canary Deployment Example:**
```python
registry = ParserRegistry("postgresql://localhost/parsers")

# Step 1: Register new parser version (0% rollout)
registry.register_parser(
    source_type="bofa_pdf",
    version="2.0.0",
    class_name="parsers.bofa_pdf_v2.BofAPDFParserV2",
    description="Improved BoFA parser with OCR fallback",
    rollout_percentage=0
)

# Step 2: Start canary with 5% traffic
registry.set_canary_rollout("bofa_pdf", "2.0.0", 5)

# Monitor health for 24 hours...

# Step 3: Expand canary to 25%
registry.set_canary_rollout("bofa_pdf", "2.0.0", 25)

# Monitor for 48 hours...

# Step 4: Promote to stable (100%)
registry.set_canary_rollout("bofa_pdf", "2.0.0", 100)

# Step 5: Deprecate old version
registry.set_canary_rollout("bofa_pdf", "1.5.0", 0)
```

**Emergency Rollback:**
```python
# Health check shows v2.0 has 15% failure rate (v1.5 had 2%)
health = registry.get_health_metrics("bofa_pdf")
# [
#   {"version": "2.0.0", "success_rate": 85.3, "failure_count": 1200},
#   {"version": "1.5.0", "success_rate": 98.1, "failure_count": 80}
# ]

# Emergency rollback to v1.5
registry.rollback("bofa_pdf", "1.5.0")
# All traffic instantly switches to v1.5.0
```

**A/B Testing:**
```python
# Assign 50% of tenants to new parser for comparison
tenants = get_all_tenant_ids()
for i, tenant_id in enumerate(tenants):
    if i % 2 == 0:
        registry.assign_tenant_override(tenant_id, "bofa_pdf", "2.0.0")
    else:
        registry.assign_tenant_override(tenant_id, "bofa_pdf", "1.5.0")

# After 1 week, compare health metrics
health_v2 = registry.get_health_metrics("bofa_pdf")
# Decide which version performs better
```

**Testing:**
- ✅ Unit tests for all registry methods
- ✅ Integration tests with test database
- ✅ Canary rollout simulation (verify percentage logic)
- ✅ Health metrics calculation validation
- ✅ Rollback scenario testing

**Monitoring:**
- Grafana dashboard: Parser success rate by version
- Alert if new version success rate < 95%
- Alert if rollback triggered
- Daily report: Parser health summary

**Decision Context:**
- Platform supports 50+ banks, each with 2-3 parser versions
- New parsers released monthly
- Canary deployments reduce risk of breaking production
- Health monitoring catches regressions before full rollout
- A/B testing validates new parsers before replacing old ones

---

## Related Primitives

- **Parser**: Registered and retrieved by this registry
- **Coordinator** (Vertical 1.2): Uses registry to find parser for `source_type`

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
