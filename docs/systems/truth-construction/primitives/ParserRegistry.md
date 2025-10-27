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

### Personal Profile (30 LOC)

**Contexto del Usuario:**
Darwin tiene 1 cuenta bancaria (Bank of America). Solo necesita 1 parser (bofa_pdf). Hardcodear el parser es más simple que cargar config YAML. Si cambia de banco en el futuro, cambia 1 línea de código.

**Implementation:**
```python
# lib/parser_registry.py (Personal - 30 LOC)
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

# Usage
registry = ParserRegistry()
parser = registry.get_parser("bofa_pdf")
observations = parser.parse(pdf_path)
```

**Características Incluidas:**
- ✅ Hardcoded dict mapping (source_type → parser)
- ✅ Error handling (ParserNotFoundError)
- ✅ Supports check before parsing
- ✅ Zero configuration

**Características NO Incluidas:**
- ❌ YAML config file (YAGNI: 1 parser, hardcode is simpler)
- ❌ Dynamic registration (YAGNI: parser list never changes)
- ❌ Parser versioning (YAGNI: always use latest)
- ❌ Canary deployments (YAGNI: single user, no gradual rollout)

**Configuración:**
```python
# Hardcoded (no config needed)
PARSERS = {"bofa_pdf": BofAPDFParser()}
```

**Performance:**
- Lookup time: 0ms (dict lookup)
- Memory: 10KB (1 parser instance)

**Upgrade Triggers:**
- Si >3 bancos → Small Business (YAML config)
- Si multi-usuario → Small Business (shared config)

---

### Small Business Profile (120 LOC)

**Contexto del Usuario:**
Firma contable procesa estados de cuenta de 8 bancos diferentes. Necesitan config YAML para que non-developers puedan agregar parsers sin editar código. 8 parsers activos, pueden llegar a 15-20 eventualmente.

**Implementation:**
```python
# lib/parser_registry.py (Small Business - 120 LOC)
import yaml
from pathlib import Path
from importlib import import_module

class ParserNotFoundError(Exception):
    pass

class ParserRegistry:
    """YAML-based parser registry."""

    def __init__(self, config_path="config/parsers.yaml"):
        self.config_path = Path(config_path)
        self._parsers = {}
        self._load_config()

    def _load_config(self):
        """Load parsers from YAML config."""
        with open(self.config_path, "r") as f:
            config = yaml.safe_load(f)

        for source_type, parser_config in config["parsers"].items():
            # Dynamic import: "parsers.bofa_pdf.BofAPDFParser"
            module_path, class_name = parser_config["class"].rsplit(".", 1)
            module = import_module(module_path)
            parser_class = getattr(module, class_name)

            # Instantiate parser
            self._parsers[source_type] = parser_class()

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

    def reload(self):
        """Reload config without restarting app."""
        self._parsers = {}
        self._load_config()

# Usage
registry = ParserRegistry(config_path="config/parsers.yaml")
parser = registry.get_parser("bofa_pdf")
```

**YAML Config:**
```yaml
# config/parsers.yaml
parsers:
  bofa_pdf:
    class: "parsers.bofa_pdf.BofAPDFParser"
    description: "Bank of America PDF statements"

  chase_csv:
    class: "parsers.chase_csv.ChaseCSVParser"
    description: "Chase CSV exports"

  wells_fargo_pdf:
    class: "parsers.wells_fargo_pdf.WellsFargoPDFParser"
    description: "Wells Fargo PDF statements"

  # ... 5 more banks
```

**Características Incluidas:**
- ✅ YAML config (non-developers can add parsers)
- ✅ Dynamic import (load parser classes at runtime)
- ✅ Reload without restart
- ✅ 8 parsers (expandable to 15-20)

**Características NO Incluidas:**
- ❌ Database storage (YAML file sufficient for 20 parsers)
- ❌ Parser versioning (always use latest version)
- ❌ Canary deployments (no gradual rollout needed)
- ❌ A/B testing (no performance comparison needed)

**Configuración:**
```yaml
parser_registry:
  config_path: "config/parsers.yaml"
  auto_reload: false
```

**Performance:**
- Startup load: 50ms (8 parsers)
- Lookup: 0ms (dict cached)
- Reload: 50ms (re-import classes)

**Upgrade Triggers:**
- Si >20 parsers → Enterprise (database registry)
- Si necesitan versioning → Enterprise
- Si multi-tenant → Enterprise (per-tenant parsers)

---

### Enterprise Profile (800 LOC)

**Contexto del Usuario:**
Plataforma multi-tenant con 50+ parsers. Necesitan: versioning (v1/v2/v3 simultáneos), canary deployments (rollout gradual de nuevos parsers), A/B testing (comparar accuracy de v2 vs v3), per-tenant parser selection. Database registry con PostgreSQL.

**Implementation:**
```python
# lib/parser_registry.py (Enterprise - 800 LOC)
import psycopg2
from importlib import import_module
from datetime import datetime

class ParserNotFoundError(Exception):
    pass

class ParserRegistry:
    """Database-backed parser registry with versioning."""

    def __init__(self, db_connection_string):
        self.conn = psycopg2.connect(db_connection_string)
        self._cache = {}  # In-memory cache

    def get_parser(
        self,
        source_type: str,
        tenant_id: str = None,
        version: str = "latest"
    ):
        """
        Get parser for source_type.

        Args:
            source_type: "bofa_pdf", "chase_csv", etc.
            tenant_id: Tenant ID (for per-tenant overrides)
            version: "latest" | "v1" | "v2" | specific semver

        Returns:
            Parser instance
        """
        # Check cache
        cache_key = f"{source_type}:{tenant_id}:{version}"
        if cache_key in self._cache:
            return self._cache[cache_key]

        # Query database
        cursor = self.conn.cursor()

        # Tenant-specific override?
        if tenant_id:
            cursor.execute("""
                SELECT parser_class, version, canary_percentage
                FROM parser_registry
                WHERE source_type = %s
                  AND tenant_id = %s
                  AND enabled = true
                ORDER BY version DESC
                LIMIT 1
            """, (source_type, tenant_id))
            row = cursor.fetchone()

            # Canary deployment logic
            if row and row[2] < 100:  # canary_percentage < 100
                import random
                if random.random() * 100 < row[2]:
                    # Use canary version
                    parser_class, version, canary_pct = row
                else:
                    # Fallback to stable version
                    cursor.execute("""
                        SELECT parser_class, version
                        FROM parser_registry
                        WHERE source_type = %s
                          AND tenant_id IS NULL
                          AND enabled = true
                          AND canary_percentage = 100
                        ORDER BY version DESC
                        LIMIT 1
                    """, (source_type,))
                    row = cursor.fetchone()
                    parser_class, version = row

        # Global parser (no tenant override)
        if not row:
            cursor.execute("""
                SELECT parser_class, version
                FROM parser_registry
                WHERE source_type = %s
                  AND tenant_id IS NULL
                  AND enabled = true
                ORDER BY version DESC
                LIMIT 1
            """, (source_type,))
            row = cursor.fetchone()

        if not row:
            raise ParserNotFoundError(
                f"No parser found for source_type: {source_type}"
            )

        parser_class, version = row

        # Dynamic import
        module_path, class_name = parser_class.rsplit(".", 1)
        module = import_module(module_path)
        parser = getattr(module, class_name)()

        # Cache
        self._cache[cache_key] = parser

        return parser

    def register_parser(
        self,
        source_type: str,
        parser_class: str,
        version: str,
        tenant_id: str = None,
        canary_percentage: int = 100
    ):
        """
        Register new parser version.

        Args:
            canary_percentage: 0-100, gradual rollout (0 = disabled, 100 = full)
        """
        cursor = self.conn.cursor()
        cursor.execute("""
            INSERT INTO parser_registry (
                source_type, parser_class, version, tenant_id, canary_percentage, enabled
            ) VALUES (%s, %s, %s, %s, %s, true)
            ON CONFLICT (source_type, version, tenant_id)
            DO UPDATE SET
                parser_class = EXCLUDED.parser_class,
                canary_percentage = EXCLUDED.canary_percentage
        """, (source_type, parser_class, version, tenant_id, canary_percentage))
        self.conn.commit()

        # Invalidate cache
        self._cache = {}

# Database Schema
"""
CREATE TABLE parser_registry (
    id SERIAL PRIMARY KEY,
    source_type TEXT NOT NULL,       -- "bofa_pdf", "chase_csv"
    parser_class TEXT NOT NULL,      -- "parsers.bofa_pdf.BofAPDFParserV2"
    version TEXT NOT NULL,           -- "v2.1.0"
    tenant_id UUID,                  -- NULL = global, UUID = tenant-specific
    canary_percentage INT DEFAULT 100, -- 0-100, gradual rollout
    enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE (source_type, version, tenant_id)
);

CREATE INDEX idx_parser_registry_lookup ON parser_registry (source_type, tenant_id, enabled);
"""

# Usage - Canary deployment (gradual rollout)
registry = ParserRegistry(db_connection_string="postgresql://localhost/app")

# Register new parser v2 at 10% canary
registry.register_parser(
    source_type="bofa_pdf",
    parser_class="parsers.bofa_pdf.BofAPDFParserV2",
    version="v2.0.0",
    canary_percentage=10  # 10% of users get v2, 90% get v1
)

# After 24h monitoring, if v2 accuracy >= v1, increase to 50%
registry.register_parser(
    source_type="bofa_pdf",
    parser_class="parsers.bofa_pdf.BofAPDFParserV2",
    version="v2.0.0",
    canary_percentage=50
)

# After 48h, if no issues, full rollout
registry.register_parser(
    source_type="bofa_pdf",
    parser_class="parsers.bofa_pdf.BofAPDFParserV2",
    version="v2.0.0",
    canary_percentage=100
)

# Get parser (automatically applies canary logic)
parser = registry.get_parser("bofa_pdf", tenant_id="acme_corp")
```

**Características Incluidas:**
- ✅ PostgreSQL registry (50+ parsers, versions, tenants)
- ✅ Parser versioning (v1, v2, v3 simultáneos)
- ✅ Canary deployments (gradual rollout 10% → 50% → 100%)
- ✅ Per-tenant overrides (Acme Corp uses v3, others use v2)
- ✅ In-memory cache (fast lookups)
- ✅ A/B testing support (canary_percentage)

**Características NO Incluidas:**
- ❌ Automatic accuracy tracking (manual monitoring via metrics)
- ❌ Auto-rollback (manual decision based on dashboards)

**Configuración:**
```yaml
parser_registry:
  database:
    connection_string: "postgresql://db:5432/app"
    pool_size: 20
  cache:
    enabled: true
    ttl_seconds: 300  # 5 min
  canary:
    default_percentage: 10  # Start at 10%
```

**Performance:**
- Lookup (cached): 0ms
- Lookup (DB query): 5ms
- Registration: 10ms (DB insert + cache invalidation)
- Canary decision: <1ms (random selection)

**No Further Tiers:**
- Scale horizontally (read replicas for registry DB)

---

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
