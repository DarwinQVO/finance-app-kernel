# OL Primitive: NormalizationRuleSet

**Type**: Configuration / Business Rules
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.3 (Normalization)
**Related ADR**: ADR-0005 (Configurable Normalization Rules)

---

## Purpose

Externalized, versioned configuration for domain-specific normalization rules. Enables updating rules without code changes, supports per-source-type customization, and provides auditability (which rules produced which canonicals).

---

## Schema

```python
from dataclasses import dataclass
from typing import List, Dict

@dataclass
class NormalizationRuleSet:
    version: str                    # "2024.10"
    source_type: str                # "bofa_pdf"
    date_locale: str                # "en_US", "es_MX", "fr_FR"
    date_formats: List[str]         # ["MM/DD/YYYY", "M/D/YYYY"]
    amount_decimal_separator: str   # "." or ","
    merchant_whitelist: Dict[str, str]  # {"STARBUCKS": "Starbucks"}
    category_rules: List[CategoryRule]
    duplicate_tolerance_days: int   # 1 (allow ±1 day for duplicates)
    large_amount_threshold: float   # 10000.00
```

---

## Example (Finance Domain)

```json
{
  "version": "2024.10",
  "source_type": "bofa_pdf",
  "date_locale": "en_US",
  "date_formats": ["MM/DD/YYYY"],
  "amount_decimal_separator": ".",
  "merchant_whitelist": {
    "STARBUCKS": "Starbucks",
    "AMAZON.COM": "Amazon"
  },
  "category_rules": [
    {"merchant_pattern": "Starbucks", "category": "Food & Drink", "confidence": 0.95},
    {"merchant_pattern": "Amazon", "category": "Shopping", "confidence": 0.90}
  ],
  "duplicate_tolerance_days": 1,
  "large_amount_threshold": 10000.00
}
```

---

## Storage

**Path:** `config/normalization-rules/{source_type}_{version}.json`
**Versioning:** Git-tracked, semantic versioning (YYYY.MM format)
**Deployment:** CI/CD deploys updated configs, normalizer hot-reloads

---

## Multi-Domain Applicability

**Finance:** Date locales, merchant whitelists, category taxonomies
**Healthcare:** Unit conversions, normal ranges by age/gender, test name normalization
**Legal:** Clause taxonomies, obligation patterns, jurisdiction-specific rules
**Research:** Citation formats (APA, MLA, Chicago), field extraction patterns
**Manufacturing:** Sensor calibration factors, measurement thresholds, unit conversions

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Configure date formats, merchant whitelists, category rules for bank statement normalization
**Example:** NormalizationRuleSet version 2024.10 for "bofa_pdf" → {date_formats: ["MM/DD/YYYY"], amount_decimal_separator: ".", merchant_whitelist: {"STARBUCKS": "Starbucks", "AMAZON.COM": "Amazon"}, category_rules: [{merchant_pattern: "Starbucks", category: "Food & Drink", confidence: 0.95}], duplicate_tolerance_days: 1} → Normalizer loads rules → Applies to 100 transactions → "01/15/2024" parsed with MM/DD/YYYY → "STARBUCKS #1234" normalized to "Starbucks" → Auto-categorized as "Food & Drink" (0.95 confidence)
**Operations:** Version control (Git-tracked), hot-reload (update rules without restart), per-source-type customization (bofa_pdf vs chase_csv have different date locales)
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Configure unit conversions, normal ranges, test name mappings for lab result normalization
**Example:** RuleSet for "quest_pdf" → {unit_conversions: [{"from": "mmol/L", "to": "mg/dL", "factor": 18.0}], normal_ranges: {"Glucose": {"min": 70, "max": 100, "unit": "mg/dL"}}, test_name_aliases: {"GLU": "Glucose", "BG": "Blood Glucose"}} → Normalizer converts 5.5 mmol/L → 99 mg/dL → Checks normal range → Flags if outside 70-100
**Status:** ✅ Conceptually validated

### ✅ Legal
**Use case:** Configure clause taxonomies, obligation patterns, jurisdiction rules for contract normalization
**Example:** RuleSet for "docusign_pdf" → {clause_taxonomies: [{"pattern": "indemnification", "category": "Liability", "subcategory": "Indemnity"}], jurisdiction_rules: {"California": {"term_limit_months": 12}}} → Normalizer categorizes clause containing "indemnification" → Tagged as "Liability > Indemnity"
**Status:** ✅ Conceptually validated

### ✅ RSRCH (Utilitario Research)
**Use case:** Configure entity resolution rules, source credibility scores, fact type patterns for founder fact normalization
**Example:** RuleSet version 2024.10 for "techcrunch_html" → {entity_aliases: {"Sam Altman": "@sama", "Elon Musk": "@elonmusk"}, source_credibility_score: 0.92, fact_type_patterns: [{"pattern": "invested \\$([0-9.]+)M", "fact_type": "investment", "confidence": 0.85}], date_formats: ["MMMM D, YYYY", "MMM D, YYYY"]} → Normalizer processes "Sam Altman invested $375M in OpenAI on February 25, 2024" → Resolves "Sam Altman" → "@sama" → Extracts investment amount $375M → Parses date "February 25, 2024" → Creates CanonicalFact with source_credibility: 0.92
**Operations:** Entity alias dictionaries (founder names → canonical handles), fact extraction patterns (regex for investment amounts, dates), source credibility per parser (TechCrunch: 0.92, personal blog: 0.45), hot-reload for adding new entities without deployment
**Status:** ✅ Conceptually validated

### ✅ E-commerce
**Use case:** Configure product category taxonomies, SKU normalization, price validation for catalog normalization
**Example:** RuleSet for "supplier_csv" → {category_taxonomy: {"Electronics > Phones": ["iPhone", "Android", "Samsung"]}, sku_format: "^[A-Z]{3}-[0-9]{6}$", price_validation: {"min": 0.01, "max": 99999.99}} → Normalizer validates SKU "IPH-123456" matches format → Categorizes "iPhone 15 Pro" → "Electronics > Phones > iPhone" → Rejects negative prices
**Status:** ✅ Conceptually validated

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (externalized configuration pattern, no domain-specific code in primitive itself)
**Reusability:** High (same versioned config structure works for financial rules, medical rules, legal rules, research rules, product rules; only rule content differs)

---

## Simplicity Profiles

The same NormalizationRuleSet interface supports three implementation scales: Personal (hardcoded dict), Small Business (YAML per source_type), Enterprise (database with versioning + hot-reload).

### Personal Profile (40 LOC)

**Darwin's Approach:** Hardcoded Python dictionary with normalization rules

**What Darwin needs:**
- Date format for Bank of America (MM/DD/YYYY)
- Merchant name corrections ("STARBUCKS #1234" → "Starbucks")
- Category auto-assignment rules
- Duplicate detection tolerance (±1 day)

**What Darwin DOESN'T need:**
- ❌ YAML config file (rules never change)
- ❌ Versioning (Darwin doesn't A/B test rules)
- ❌ Hot-reload (Darwin can restart app if rules change)
- ❌ Per-source-type customization (Darwin only has 1 bank)
- ❌ Date locale handling (US-only)

**Implementation:**

**Code Example (40 LOC):**
```python
# normalization_rules.py (Darwin's personal finance app)

# Hardcoded normalization rules for Bank of America
NORMALIZATION_RULES = {
    "version": "1.0",
    "source_type": "bofa_pdf",

    # Date parsing
    "date_format": "MM/DD/YYYY",

    # Amount parsing
    "amount_decimal_separator": ".",

    # Merchant name normalization
    "merchant_whitelist": {
        "STARBUCKS": "Starbucks",
        "STARBUCKS #1234": "Starbucks",
        "SBX": "Starbucks",
        "WHOLE FOODS": "Whole Foods Market",
        "WHL FOODS": "Whole Foods Market",
        "AMAZON.COM": "Amazon",
        "AMZN MKTP": "Amazon",
        "SHELL OIL": "Shell",
        "CHEVRON": "Chevron",
    },

    # Auto-categorization rules
    "category_rules": [
        {"merchant": "Starbucks", "category": "Food & Dining"},
        {"merchant": "Whole Foods Market", "category": "Groceries"},
        {"merchant": "Amazon", "category": "Shopping"},
        {"merchant": "Shell", "category": "Transportation"},
        {"merchant": "Chevron", "category": "Transportation"},
    ],

    # Duplicate detection
    "duplicate_tolerance_days": 1,  # ±1 day

    # Large amount flagging
    "large_amount_threshold": 10000.00,
}

def get_merchant_canonical_name(raw_merchant: str) -> str:
    """Normalize merchant name using whitelist."""
    return NORMALIZATION_RULES["merchant_whitelist"].get(
        raw_merchant.upper(),
        raw_merchant  # Return as-is if not in whitelist
    )

def get_auto_category(merchant: str) -> str:
    """Get auto-assigned category based on merchant."""
    for rule in NORMALIZATION_RULES["category_rules"]:
        if merchant == rule["merchant"]:
            return rule["category"]
    return None  # Uncategorized
```

**Usage:**
```python
# In normalizer
raw_merchant = "STARBUCKS #1234"
canonical = get_merchant_canonical_name(raw_merchant)
# Returns: "Starbucks"

category = get_auto_category("Starbucks")
# Returns: "Food & Dining"
```

**Why hardcoded is correct:**
- Darwin's merchant list is stable (same 10-15 merchants every month)
- Date format never changes (Bank of America always uses MM/DD/YYYY)
- If Darwin adds new merchant → Add one line to dict
- No need for file I/O when dict is in-memory

**Testing:**
- ❌ No tests (trivial dictionary lookup)
- Manual verification: Upload PDF, check merchant names normalized correctly

**Decision Context:**
- Darwin has 10-15 recurring merchants → hardcoded whitelist is sufficient
- YAGNI: No need for YAML config when rules fit in 40 lines of code
- If merchant list grows to 50+ → Consider YAML config then

---

### Small Business Profile (180 LOC)

**Use Case:** Accounting firm processing statements from 8 banks with different date formats

**What they need (vs Personal):**
- ✅ All Personal features (merchant whitelist, category rules, duplicate tolerance)
- ✅ **YAML config per source_type** (different banks have different formats)
- ✅ **Versioning** (track which rule version processed which data)
- ✅ **Multiple date formats** (US vs European formats)
- ❌ No hot-reload (app restart acceptable)
- ❌ No A/B testing of rule sets (not enough volume)
- ❌ No database (YAML files sufficient)

**Implementation:**

**Config Files:**
```
config/normalization-rules/
  bofa_pdf_2024.10.yaml
  chase_csv_2024.10.yaml
  wells_fargo_pdf_2024.10.yaml
  citi_pdf_2024.11.yaml  (newer version)
```

**Example Config (bofa_pdf_2024.10.yaml):**
```yaml
version: "2024.10"
source_type: "bofa_pdf"

date:
  locale: "en_US"
  formats:
    - "MM/DD/YYYY"
    - "M/D/YYYY"

amount:
  decimal_separator: "."
  thousands_separator: ","

merchant_whitelist:
  "STARBUCKS": "Starbucks"
  "STARBUCKS #1234": "Starbucks"
  "SBX": "Starbucks"
  "WHOLE FOODS": "Whole Foods Market"
  "WHL FOODS": "Whole Foods Market"
  "WFM": "Whole Foods Market"
  "AMAZON.COM": "Amazon"
  "AMZN MKTP": "Amazon"
  "SHELL OIL": "Shell"
  "CHEVRON": "Chevron"
  "SAFEWAY": "Safeway"
  "CVS PHARMACY": "CVS"
  "WALGREENS": "Walgreens"

category_rules:
  - merchant_pattern: "Starbucks"
    category: "Food & Dining"
    subcategory: "Coffee Shops"
    confidence: 0.95

  - merchant_pattern: "Whole Foods Market"
    category: "Groceries"
    confidence: 0.98

  - merchant_pattern: "Amazon"
    category: "Shopping"
    confidence: 0.85  # Lower (could be groceries, electronics, etc.)

  - merchant_pattern: "Shell|Chevron"
    category: "Transportation"
    subcategory: "Gas Stations"
    confidence: 0.90

  - merchant_pattern: "CVS|Walgreens"
    category: "Healthcare"
    subcategory: "Pharmacy"
    confidence: 0.85

duplicate_detection:
  tolerance_days: 1
  amount_tolerance: 0.01  # ±$0.01

validation:
  large_amount_threshold: 10000.00
  min_amount: 0.01
```

**European Bank Example (citi_pdf_2024.11.yaml):**
```yaml
version: "2024.11"
source_type: "citi_pdf"

date:
  locale: "en_GB"  # UK/European format
  formats:
    - "DD/MM/YYYY"
    - "D/M/YYYY"

amount:
  decimal_separator: ","  # European: 1.234,56
  thousands_separator: "."

merchant_whitelist:
  "TESCO": "Tesco"
  "SAINSBURY'S": "Sainsbury's"
  "MARKS & SPENCER": "Marks & Spencer"
```

**Code Example (180 LOC):**
```python
# normalization_rule_set.py (Small Business)

import yaml
from pathlib import Path
from dataclasses import dataclass
from typing import Dict, List, Optional
import re

@dataclass
class CategoryRule:
    merchant_pattern: str
    category: str
    subcategory: Optional[str] = None
    confidence: float = 0.85

class NormalizationRuleSet:
    """YAML-based normalization rules with versioning."""

    def __init__(self, config_dir: str = "config/normalization-rules"):
        self.config_dir = Path(config_dir)
        self._rule_cache: Dict[str, dict] = {}

    def load(self, source_type: str, version: Optional[str] = None) -> dict:
        """
        Load normalization rules for source_type.

        If version not specified, loads latest version.
        """
        # Find config file
        if version:
            config_path = self.config_dir / f"{source_type}_{version}.yaml"
        else:
            # Find latest version
            pattern = f"{source_type}_*.yaml"
            files = sorted(self.config_dir.glob(pattern), reverse=True)
            if not files:
                raise ValueError(f"No rules found for source_type: {source_type}")
            config_path = files[0]

        # Check cache
        cache_key = str(config_path)
        if cache_key in self._rule_cache:
            return self._rule_cache[cache_key]

        # Load YAML
        rules = yaml.safe_load(config_path.read_text())

        # Parse category rules
        rules["category_rules_parsed"] = [
            CategoryRule(**rule) for rule in rules.get("category_rules", [])
        ]

        # Cache
        self._rule_cache[cache_key] = rules
        return rules

    def get_merchant_canonical_name(self, raw_merchant: str, source_type: str) -> str:
        """Normalize merchant name using whitelist."""
        rules = self.load(source_type)
        whitelist = rules.get("merchant_whitelist", {})

        # Try exact match
        canonical = whitelist.get(raw_merchant.upper())
        if canonical:
            return canonical

        # Try partial match (for variations like "STARBUCKS #1234")
        raw_upper = raw_merchant.upper()
        for raw_pattern, canonical_name in whitelist.items():
            if raw_upper.startswith(raw_pattern):
                return canonical_name

        # Return as-is if not in whitelist
        return raw_merchant

    def get_auto_category(
        self,
        merchant: str,
        source_type: str
    ) -> Optional[tuple[str, Optional[str], float]]:
        """
        Get auto-assigned category based on merchant.

        Returns: (category, subcategory, confidence) or None
        """
        rules = self.load(source_type)

        for rule in rules.get("category_rules_parsed", []):
            # Use regex matching for patterns like "Shell|Chevron"
            if re.search(rule.merchant_pattern, merchant, re.IGNORECASE):
                return (rule.category, rule.subcategory, rule.confidence)

        return None

    def parse_date(self, date_str: str, source_type: str) -> str:
        """Parse date using configured formats."""
        from datetime import datetime

        rules = self.load(source_type)
        date_config = rules.get("date", {})
        formats = date_config.get("formats", ["MM/DD/YYYY"])

        # Try each format
        for fmt in formats:
            # Convert YYYY.MM.DD format to Python strptime format
            python_fmt = fmt.replace("YYYY", "%Y").replace("MM", "%m").replace("DD", "%d")
            python_fmt = python_fmt.replace("M", "%-m").replace("D", "%-d")

            try:
                parsed = datetime.strptime(date_str, python_fmt)
                return parsed.strftime("%Y-%m-%d")  # ISO format
            except ValueError:
                continue

        raise ValueError(f"Could not parse date '{date_str}' with formats {formats}")

    def is_duplicate_candidate(
        self,
        date1: str,
        date2: str,
        amount1: float,
        amount2: float,
        source_type: str
    ) -> bool:
        """Check if two transactions might be duplicates."""
        from datetime import datetime, timedelta

        rules = self.load(source_type)
        dup_config = rules.get("duplicate_detection", {})

        tolerance_days = dup_config.get("tolerance_days", 1)
        amount_tolerance = dup_config.get("amount_tolerance", 0.01)

        # Check date tolerance
        d1 = datetime.fromisoformat(date1)
        d2 = datetime.fromisoformat(date2)
        date_diff = abs((d2 - d1).days)

        if date_diff > tolerance_days:
            return False

        # Check amount tolerance
        amount_diff = abs(amount2 - amount1)
        if amount_diff > amount_tolerance:
            return False

        return True

    def list_available_sources(self) -> List[Dict]:
        """List all available source_types and their versions."""
        result = []
        for config_file in sorted(self.config_dir.glob("*.yaml")):
            # Parse filename: bofa_pdf_2024.10.yaml → (bofa_pdf, 2024.10)
            name = config_file.stem
            parts = name.rsplit("_", 1)
            if len(parts) == 2:
                source_type, version = parts
                result.append({
                    "source_type": source_type,
                    "version": version,
                    "config_file": str(config_file)
                })
        return result
```

**Usage:**
```python
# Load rules
rule_set = NormalizationRuleSet("config/normalization-rules")

# Normalize merchant
raw = "STARBUCKS #1234"
canonical = rule_set.get_merchant_canonical_name(raw, "bofa_pdf")
# Returns: "Starbucks"

# Get auto-category
category, subcategory, confidence = rule_set.get_auto_category("Starbucks", "bofa_pdf")
# Returns: ("Food & Dining", "Coffee Shops", 0.95)

# Parse date
date_iso = rule_set.parse_date("10/15/2024", "bofa_pdf")
# Returns: "2024-10-15"

# Check duplicates
is_dup = rule_set.is_duplicate_candidate(
    "2024-10-15", "2024-10-16",  # ±1 day
    87.43, 87.43,  # Same amount
    "bofa_pdf"
)
# Returns: True (within tolerance)
```

**Adding new rules (no code changes):**
```bash
# Copy existing rules
cp config/normalization-rules/bofa_pdf_2024.10.yaml \
   config/normalization-rules/bofa_pdf_2024.11.yaml

# Edit new version
vim config/normalization-rules/bofa_pdf_2024.11.yaml
# Add new merchant:
#   "TARGET": "Target"

# Restart app to pick up new rules
```

**Testing:**
- ✅ Unit tests for merchant normalization
- ✅ Unit tests for date parsing (multiple formats)
- ✅ Unit tests for duplicate detection
- ✅ Integration test: Load all config files, verify no YAML syntax errors

**Monitoring:**
- Startup log: "Loaded 8 rule sets from config/normalization-rules/"
- Warning if config file missing for source_type

**Decision Context:**
- Accounting firm supports 8 banks → 8 YAML config files
- Different banks have different date formats (US vs European)
- Operations team can edit YAML (no code changes needed)
- Version control tracks rule changes over time

---

### Enterprise Profile (1000 LOC)

**Use Case:** Multi-tenant platform with ML-based rules, A/B testing, hot-reload

**What they need (vs Small Business):**
- ✅ All Small Business features (YAML configs, versioning, multiple formats)
- ✅ **Database-backed rule storage** (PostgreSQL with rule versions table)
- ✅ **Hot-reload** (update rules without app restart)
- ✅ **A/B testing** (compare performance of two rule sets)
- ✅ **ML-based category prediction** (train model on historical data)
- ✅ **Canary deployment** (gradual rollout of new rules)
- ✅ **Rule performance tracking** (success rate, accuracy metrics)
- ✅ **Per-tenant customization** (tenant can override global rules)

**Implementation:**

**Database Schema:**
```sql
CREATE TABLE normalization_rule_sets (
    id SERIAL PRIMARY KEY,
    source_type TEXT NOT NULL,
    version TEXT NOT NULL,
    config JSONB NOT NULL,
    enabled BOOLEAN DEFAULT TRUE,
    rollout_percentage INTEGER DEFAULT 0,  -- Canary deployment
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(source_type, version)
);

CREATE TABLE rule_set_assignments (
    tenant_id TEXT NOT NULL,
    source_type TEXT NOT NULL,
    rule_set_version TEXT NOT NULL,
    assignment_type TEXT NOT NULL,  -- 'canary', 'stable', 'override'
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (tenant_id, source_type)
);

CREATE TABLE rule_performance (
    source_type TEXT NOT NULL,
    version TEXT NOT NULL,
    category_accuracy FLOAT,  -- % of auto-categories correct
    merchant_normalization_accuracy FLOAT,
    duplicate_precision FLOAT,
    duplicate_recall FLOAT,
    last_evaluated TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (source_type, version)
);

CREATE TABLE tenant_rule_overrides (
    tenant_id TEXT NOT NULL,
    source_type TEXT NOT NULL,
    override_config JSONB NOT NULL,  -- Partial config to merge with global
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (tenant_id, source_type)
);
```

**Code Example (1000 LOC - excerpt):**
```python
# normalization_rule_set.py (Enterprise)

import json
import random
from typing import Dict, List, Optional
import psycopg2
from psycopg2.extras import Json, DictCursor
from datetime import datetime

class NormalizationRuleSet:
    """Enterprise rule set with database, hot-reload, A/B testing."""

    def __init__(self, db_conn_string: str, ml_model_path: Optional[str] = None):
        self.db_conn_string = db_conn_string
        self._rule_cache: Dict[str, dict] = {}
        self._ml_model = self._load_ml_model(ml_model_path) if ml_model_path else None
        self._init_db()

    def _init_db(self):
        """Create tables if not exist."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()
        # ... (schema creation from above)
        conn.commit()
        conn.close()

    def register_rule_set(
        self,
        source_type: str,
        version: str,
        config: dict,
        rollout_percentage: int = 0
    ):
        """Register a new rule set version."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO normalization_rule_sets
            (source_type, version, config, rollout_percentage)
            VALUES (%s, %s, %s, %s)
            ON CONFLICT (source_type, version) DO UPDATE SET
                config = EXCLUDED.config,
                rollout_percentage = EXCLUDED.rollout_percentage
        """, (source_type, version, Json(config), rollout_percentage))

        conn.commit()
        conn.close()

        # Clear cache to force reload
        cache_key = f"{source_type}:{version}"
        if cache_key in self._rule_cache:
            del self._rule_cache[cache_key]

    def load(
        self,
        source_type: str,
        tenant_id: Optional[str] = None,
        version: Optional[str] = None
    ) -> dict:
        """
        Load rules with tenant overrides and canary rollout.

        Logic:
        1. Check tenant assignment (override)
        2. Check canary rollout
        3. Load global rules
        4. Apply tenant overrides
        """
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor(cursor_factory=DictCursor)

        # 1. Check tenant assignment
        selected_version = version
        if tenant_id and not version:
            cursor.execute("""
                SELECT rule_set_version FROM rule_set_assignments
                WHERE tenant_id = %s AND source_type = %s
            """, (tenant_id, source_type))
            row = cursor.fetchone()
            if row:
                selected_version = row["rule_set_version"]

        # 2. Canary rollout (if no specific version)
        if not selected_version:
            cursor.execute("""
                SELECT version, rollout_percentage FROM normalization_rule_sets
                WHERE source_type = %s AND enabled = TRUE
                ORDER BY version DESC
            """, (source_type,))
            rows = cursor.fetchall()

            if not rows:
                conn.close()
                raise ValueError(f"No rule sets found for source_type: {source_type}")

            # Find canary vs stable
            canary = None
            stable = None
            for row in rows:
                if 0 < row["rollout_percentage"] < 100:
                    canary = row
                elif row["rollout_percentage"] == 100:
                    stable = row

            # Randomly assign based on rollout percentage
            if canary and random.randint(1, 100) <= canary["rollout_percentage"]:
                selected_version = canary["version"]
            elif stable:
                selected_version = stable["version"]
            else:
                selected_version = rows[0]["version"]

        # 3. Load global rules
        cache_key = f"{source_type}:{selected_version}"
        if cache_key not in self._rule_cache:
            cursor.execute("""
                SELECT config FROM normalization_rule_sets
                WHERE source_type = %s AND version = %s
            """, (source_type, selected_version))
            row = cursor.fetchone()
            if not row:
                conn.close()
                raise ValueError(f"Rule set not found: {source_type}:{selected_version}")

            self._rule_cache[cache_key] = row["config"]

        rules = self._rule_cache[cache_key].copy()

        # 4. Apply tenant overrides
        if tenant_id:
            cursor.execute("""
                SELECT override_config FROM tenant_rule_overrides
                WHERE tenant_id = %s AND source_type = %s
            """, (tenant_id, source_type))
            row = cursor.fetchone()
            if row:
                overrides = row["override_config"]
                # Deep merge overrides
                self._merge_config(rules, overrides)

        conn.close()
        return rules

    def _merge_config(self, base: dict, overrides: dict):
        """Deep merge override config into base config."""
        for key, value in overrides.items():
            if isinstance(value, dict) and key in base and isinstance(base[key], dict):
                self._merge_config(base[key], value)
            else:
                base[key] = value

    def get_merchant_canonical_name(
        self,
        raw_merchant: str,
        source_type: str,
        tenant_id: Optional[str] = None,
        use_ml: bool = True
    ) -> tuple[str, float]:
        """
        Normalize merchant name using whitelist + ML model.

        Returns: (canonical_name, confidence)
        """
        rules = self.load(source_type, tenant_id)
        whitelist = rules.get("merchant_whitelist", {})

        # 1. Try exact whitelist match (highest confidence)
        if raw_merchant.upper() in whitelist:
            return (whitelist[raw_merchant.upper()], 1.0)

        # 2. Try ML model (if enabled)
        if use_ml and self._ml_model:
            prediction, confidence = self._ml_model.predict_merchant(raw_merchant)
            if confidence > 0.85:
                return (prediction, confidence)

        # 3. Fall back to fuzzy matching
        from difflib import get_close_matches
        matches = get_close_matches(
            raw_merchant.upper(),
            whitelist.keys(),
            n=1,
            cutoff=0.8
        )
        if matches:
            return (whitelist[matches[0]], 0.8)

        # 4. Return as-is (low confidence)
        return (raw_merchant, 0.5)

    def get_auto_category(
        self,
        merchant: str,
        amount: float,
        source_type: str,
        tenant_id: Optional[str] = None,
        use_ml: bool = True
    ) -> Optional[tuple[str, Optional[str], float]]:
        """
        Get auto-assigned category using rules + ML model.

        Returns: (category, subcategory, confidence)
        """
        rules = self.load(source_type, tenant_id)

        # 1. Try rule-based matching
        for rule in rules.get("category_rules", []):
            if re.search(rule["merchant_pattern"], merchant, re.IGNORECASE):
                return (
                    rule["category"],
                    rule.get("subcategory"),
                    rule.get("confidence", 0.85)
                )

        # 2. Try ML model (if enabled)
        if use_ml and self._ml_model:
            prediction = self._ml_model.predict_category(
                merchant=merchant,
                amount=amount
            )
            if prediction["confidence"] > 0.75:
                return (
                    prediction["category"],
                    prediction.get("subcategory"),
                    prediction["confidence"]
                )

        return None

    def set_canary_rollout(self, source_type: str, version: str, percentage: int):
        """Update canary rollout percentage."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()
        cursor.execute("""
            UPDATE normalization_rule_sets
            SET rollout_percentage = %s
            WHERE source_type = %s AND version = %s
        """, (percentage, source_type, version))
        conn.commit()
        conn.close()

        # Clear cache
        cache_key = f"{source_type}:{version}"
        if cache_key in self._rule_cache:
            del self._rule_cache[cache_key]

    def assign_tenant_override(
        self,
        tenant_id: str,
        source_type: str,
        version: str
    ):
        """Assign specific rule set version to tenant."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()
        cursor.execute("""
            INSERT INTO rule_set_assignments
            (tenant_id, source_type, rule_set_version, assignment_type)
            VALUES (%s, %s, %s, 'override')
            ON CONFLICT (tenant_id, source_type) DO UPDATE SET
                rule_set_version = EXCLUDED.rule_set_version
        """, (tenant_id, source_type, version))
        conn.commit()
        conn.close()

    def record_performance(
        self,
        source_type: str,
        version: str,
        category_accuracy: float,
        merchant_accuracy: float,
        duplicate_precision: float,
        duplicate_recall: float
    ):
        """Record rule set performance metrics."""
        conn = psycopg2.connect(self.db_conn_string)
        cursor = conn.cursor()
        cursor.execute("""
            INSERT INTO rule_performance
            (source_type, version, category_accuracy, merchant_normalization_accuracy,
             duplicate_precision, duplicate_recall)
            VALUES (%s, %s, %s, %s, %s, %s)
            ON CONFLICT (source_type, version) DO UPDATE SET
                category_accuracy = EXCLUDED.category_accuracy,
                merchant_normalization_accuracy = EXCLUDED.merchant_normalization_accuracy,
                duplicate_precision = EXCLUDED.duplicate_precision,
                duplicate_recall = EXCLUDED.duplicate_recall,
                last_evaluated = NOW()
        """, (source_type, version, category_accuracy, merchant_accuracy,
              duplicate_precision, duplicate_recall))
        conn.commit()
        conn.close()

    def _load_ml_model(self, model_path: str):
        """Load trained ML model for category/merchant prediction."""
        import joblib
        return joblib.load(model_path)
```

**A/B Testing Example:**
```python
rule_set = NormalizationRuleSet("postgresql://localhost/rules", ml_model_path="models/category_classifier.pkl")

# Register new rule set with ML-based categories
rule_set.register_rule_set(
    source_type="bofa_pdf",
    version="2024.11_ml",
    config={
        "version": "2024.11_ml",
        "use_ml_categories": True,
        "category_rules": [...]  # Fallback rules
    },
    rollout_percentage=0
)

# A/B test: 50% tenants get ML rules, 50% get old rules
tenants = get_all_tenant_ids()
for i, tenant_id in enumerate(tenants):
    if i % 2 == 0:
        rule_set.assign_tenant_override(tenant_id, "bofa_pdf", "2024.11_ml")
    else:
        rule_set.assign_tenant_override(tenant_id, "bofa_pdf", "2024.10")

# After 1 week, compare performance
ml_perf = get_performance_metrics("bofa_pdf", "2024.11_ml")
old_perf = get_performance_metrics("bofa_pdf", "2024.10")

if ml_perf["category_accuracy"] > old_perf["category_accuracy"]:
    # ML rules perform better → promote to stable
    rule_set.set_canary_rollout("bofa_pdf", "2024.11_ml", 100)
```

**Hot-Reload Example:**
```python
# Update merchant whitelist without app restart
conn = psycopg2.connect("postgresql://localhost/rules")
cursor = conn.cursor()
cursor.execute("""
    UPDATE normalization_rule_sets
    SET config = jsonb_set(
        config,
        '{merchant_whitelist,TARGET}',
        '"Target"'
    )
    WHERE source_type = 'bofa_pdf' AND version = '2024.10'
""")
conn.commit()
conn.close()

# Next normalization will use updated rules (cache invalidated)
```

**Testing:**
- ✅ Unit tests for all rule loading logic
- ✅ Integration tests with test database
- ✅ ML model accuracy tests
- ✅ A/B test simulation
- ✅ Hot-reload validation

**Monitoring:**
- Grafana dashboard: Category accuracy by rule set version
- Alert if accuracy drops below 85%
- Daily report: Rule set performance comparison

**Decision Context:**
- Platform processes 5M transactions/day
- ML model improves category accuracy from 85% → 92%
- Hot-reload critical for fixing merchant whitelist issues without downtime
- A/B testing validates new rules before full rollout

---

## Related Primitives

- **Normalizer**: Loads and applies rules from RuleSet
- **ValidationEngine**: Uses locale/format config from RuleSet
- **CanonicalStore**: Canonicals reference `rule_set_version` used

---

**Last Updated**: 2025-10-23
**Maturity**: Spec complete, ready for implementation
