# OL Primitive: Normalizer

**Type**: Data Transformation / Validation
**Domain**: Universal (domain-agnostic)
**Version**: 1.0
**Status**: Specification
**Introduced in**: Vertical 1.3 (Normalization)

---

## Purpose

Universal interface for transforming raw observations into validated canonical records. Normalizers apply domain-specific validation rules (dates, amounts, business logic) to produce clean, validated data ready for analysis.

---

## Simplicity Profiles

**This section shows how the same universal normalizer interface is configured differently based on usage scale.**

### Profile 1: Personal Use (Finance App - Hardcoded Rules, No ML)

**Context:** Darwin's transactions (500/year), simple rules for cleaning merchants and parsing dates/amounts

**Configuration:**
```yaml
normalizer:
  version: "1.0.0"
  rules:
    type: "hardcoded"  # Python dict, not external file
    merchant_normalization:
      - pattern: "WHOLE FOODS.*" → "Whole Foods Market"
      - pattern: "STARBUCKS.*" → "Starbucks"
      # ... 20 total rules
    category_assignment:
      - merchant_contains: "Whole Foods" → "Groceries"
      - merchant_contains: "Starbucks" → "Dining"
      # ... 15 total rules
  validation:
    date_format: "MM/DD/YYYY"  # BoFA format
    amount_validation: false  # Accept any number
    required_fields: [date, amount, merchant]
  ml_features: false  # No machine learning
  confidence_scoring: false  # All rules = 1.0 confidence
```

**What's Used:**
- ✅ Basic `normalize()` method - Transform observation → canonical
- ✅ Hardcoded rules - Python dict with 20 merchant patterns
- ✅ String cleaning - Strip whitespace, uppercase, remove trailing numbers
- ✅ Date parsing - Assume MM/DD/YYYY format (BoFA-specific)
- ✅ Amount parsing - Remove "$" and ",", convert to float
- ✅ Simple category assignment - Keyword matching (15 categories)

**What's NOT Used:**
- ❌ Rule engine - No external rules file, hardcoded in Python
- ❌ Machine learning - No ML model for merchant normalization
- ❌ Confidence scoring - All normalizations = 1.0 confidence
- ❌ Validation warnings - Either passes or fails (no warnings)
- ❌ Multi-currency support - USD only
- ❌ Fuzzy matching - Exact string matching
- ❌ External APIs - No merchant data enrichment
- ❌ Rule versioning - Single version (1.0.0), never changes

**Implementation Complexity:** **LOW**
- ~150 lines Python
- Hardcoded dicts for rules
- No dependencies (stdlib only)
- No tests (manual verification)

**Narrative Example:**
> Normalizer processes observation from BoFA statement:
> ```python
> observation = Observation(
>     upload_id="UL_abc123",
>     row_id=0,
>     raw_data={
>         "date": "10/15/2024",
>         "description": "WHOLE FOODS MARKET #1234",
>         "amount": "-$87.43",
>         "balance": "$4,462.89"
>     }
> )
>
> # Normalization steps:
> normalizer = SimpleFinanceNormalizer()
> result = normalizer.normalize(observation)
>
> # Step 1: Parse date
> date_str = observation.raw_data["date"]  # "10/15/2024"
> date = datetime.strptime(date_str, "%m/%d/%Y")  # datetime.date(2024, 10, 15)
>
> # Step 2: Parse amount
> amount_str = observation.raw_data["amount"]  # "-$87.43"
> amount = float(amount_str.replace("$", "").replace(",", ""))  # -87.43
>
> # Step 3: Clean merchant
> merchant_raw = observation.raw_data["description"]  # "WHOLE FOODS MARKET #1234"
> merchant_clean = clean_merchant(merchant_raw)
> # - Strip whitespace: "WHOLE FOODS MARKET #1234"
> # - Remove trailing numbers: "WHOLE FOODS MARKET"
> # - Apply rule "WHOLE FOODS.*" → "Whole Foods Market"
> merchant = "Whole Foods Market"
>
> # Step 4: Assign category
> category = assign_category(merchant)
> # - Check if "Whole Foods" in merchant → Yes
> # - Return "Groceries"
>
> # Step 5: Create canonical
> canonical = CanonicalTransaction(
>     canonical_id="CT_abc123_0",
>     upload_id="UL_abc123",
>     observation_id="UL_abc123:0",
>     date=date(2024, 10, 15),
>     amount=-87.43,
>     merchant="Whole Foods Market",
>     category="Groceries",
>     account="Bank of America Checking"
> )
>
> return NormalizationResult(
>     canonical=canonical,
>     confidence=1.0,  # Always 1.0 (no confidence scoring)
>     applied_rules=["clean_merchant", "assign_category"],
>     warnings=[],
>     metadata={}
> )
> ```
>
> Result stored in CanonicalStore. If Darwin later updates rules (e.g., "Whole Foods" → "Groceries & Household"), re-run normalizer → Re-upsert with updated category → Old category overwritten.

---

### Profile 2: Small Business (Accounting Firm - Rule Engine, Validation Warnings)

**Context:** 50 clients with different expense categorization rules, need configurable rules per client

**Configuration:**
```yaml
normalizer:
  version: "2.0.0"
  rules:
    type: "yaml_file"  # External rules file per client
    rule_directory: "/var/accounting-data/normalization-rules/"
    rule_files:
      - client_abc.yaml  # Custom rules for Client ABC
      - client_xyz.yaml  # Custom rules for Client XYZ
      # ... 50 clients
  validation:
    date_formats: ["MM/DD/YYYY", "DD/MM/YYYY", "YYYY-MM-DD"]  # Multi-format
    amount_validation: true  # Validate amount is numeric
    amount_range: [-1000000, 1000000]  # Detect outliers
    required_fields: [date, amount, merchant]
    warnings_enabled: true  # Non-fatal validation warnings
  merchant_normalization:
    fuzzy_matching:
      enabled: true
      threshold: 0.85  # Levenshtein similarity
    database: "merchant_aliases.db"  # Pre-populated aliases
  confidence_scoring:
    enabled: true
    factors: [fuzzy_match_score, rule_match_count]
  audit:
    log_normalizations: true
    log_level: "info"
```

**What's Used:**
- ✅ Rule engine - Load rules from YAML files (per client)
- ✅ Multi-format date parsing - Try 3 formats, pick first that works
- ✅ Amount validation - Check numeric, within range
- ✅ Validation warnings - Non-fatal issues (e.g., "Unusual amount: $12,345")
- ✅ Fuzzy matching - Levenshtein distance for merchant names
- ✅ Merchant alias database - Pre-populated mappings
- ✅ Confidence scoring - 0.0-1.0 based on match quality
- ✅ Audit logging - Log every normalization (for client transparency)

**What's NOT Used:**
- ❌ Machine learning - Still rule-based
- ❌ External APIs - No merchant data enrichment
- ❌ Multi-currency - USD only
- ❌ Advanced ML features - No feature engineering

**Implementation Complexity:** **MEDIUM**
- ~400 lines Python
- YAML rule parser
- Levenshtein distance algorithm (python-Levenshtein package)
- SQLite merchant alias database
- Unit tests (test fuzzy matching, rule loading, warnings)

**Narrative Example:**
> Accountant uploads Client ABC expenses. Normalizer loads `client_abc.yaml`:
> ```yaml
> merchant_rules:
>   - pattern: "AMZN.*" → "Amazon"
>   - pattern: "SQ .*" → "Square Payment"
>   - fuzzy: "Starbuks" → "Starbucks"  # Typo tolerance
> category_rules:
>   - merchant: "Amazon" → "Office Supplies" if amount < 100 else "Equipment"
>   - merchant: "Starbucks" → "Meals & Entertainment"
> ```
>
> Observation: `{"date": "15/10/2024", "description": "Starbuks Coffee", "amount": "$5.75"}`
>
> Normalization:
> 1. Parse date: Try "MM/DD/YYYY" → Fails (15 > 12). Try "DD/MM/YYYY" → Success (date(2024, 10, 15))
> 2. Parse amount: "$5.75" → 5.75 ✅
> 3. Validate amount: 5.75 in range [-1M, 1M] ✅
> 4. Clean merchant: "Starbuks Coffee" → Query fuzzy matcher
>    - Check similarity to "Starbucks": Levenshtein("Starbuks", "Starbucks") = 0.88 > 0.85 threshold ✅
>    - Map to "Starbucks" (confidence = 0.88)
> 5. Assign category: "Starbucks" → "Meals & Entertainment"
> 6. Warning: Amount < $10 (unusually small for Meals) → Add warning: "Small meal expense ($5.75)"
>
> Result:
> ```python
> NormalizationResult(
>     canonical=CanonicalTransaction(
>         date=date(2024, 10, 15),
>         amount=-5.75,
>         merchant="Starbucks",
>         category="Meals & Entertainment"
>     ),
>     confidence=0.88,  # From fuzzy match
>     applied_rules=["fuzzy_merchant_match", "category_meal"],
>     warnings=["Small meal expense ($5.75)"],
>     metadata={"fuzzy_match_score": 0.88, "original_merchant": "Starbuks Coffee"}
> )
> ```
>
> Accountant reviews normalization log → Sees warning → Confirms $5.75 is correct (coffee only) → Proceeds.

---

### Profile 3: Enterprise (Bank - ML Model, Multi-Currency, External APIs)

**Context:** 10M transactions/month, need ML-based merchant normalization, multi-currency support, external data enrichment

**Configuration:**
```yaml
normalizer:
  version: "3.5.0"
  rules:
    type: "hybrid"  # Rules + ML model
    rule_engine: "drools"  # Java-based rule engine
    rule_versioning: true  # Track rule changes
    rule_deployment: "canary"  # Test new rules on 5% traffic
  ml_model:
    enabled: true
    model_path: "models/merchant_normalizer_v3.pkl"
    model_type: "random_forest"
    features: [merchant_text, amount, category_hint, time_of_day, day_of_week]
    confidence_threshold: 0.75  # Use ML if confidence > 0.75, else fallback to rules
  merchant_normalization:
    fuzzy_matching:
      enabled: true
      algorithm: "jaro_winkler"
      threshold: 0.90
    external_enrichment:
      enabled: true
      apis:
        - google_places  # Merchant address, hours, category
        - dun_bradstreet  # Business metadata
      cache_ttl_days: 30
  multi_currency:
    enabled: true
    default_currency: "USD"
    conversion_api: "openexchangerates.org"
    cache_ttl_hours: 1
  validation:
    date_formats: ["MM/DD/YYYY", "DD/MM/YYYY", "YYYY-MM-DD", "ISO8601"]
    amount_validation: true
    amount_anomaly_detection: true  # ML-based outlier detection
    pii_detection: true  # Flag SSN, credit card numbers in descriptions
  confidence_scoring:
    enabled: true
    factors:
      - ml_model_score: 0.6  # Weight 60%
      - fuzzy_match_score: 0.2  # Weight 20%
      - rule_match_count: 0.1  # Weight 10%
      - external_api_match: 0.1  # Weight 10%
  audit:
    log_all_normalizations: true
    log_destination: "elasticsearch"
    include_features: true  # Log ML features for debugging
  metrics:
    enabled: true
    backend: "prometheus"
    labels: [model_version, confidence_bucket, currency]
  performance:
    batch_size: 1000  # Batch normalization for efficiency
    parallel_workers: 16
    cache:
      enabled: true
      backend: "redis"
      ttl_hours: 24
```

**What's Used:**
- ✅ ML model - Random forest classifier for merchant normalization
- ✅ Feature engineering - Extract 5 features (text, amount, time, day)
- ✅ Hybrid rules + ML - Use ML if confidence > 0.75, else rules
- ✅ External APIs - Google Places, Dun & Bradstreet for merchant data
- ✅ Multi-currency support - Convert all amounts to USD
- ✅ Advanced fuzzy matching - Jaro-Winkler algorithm
- ✅ PII detection - Flag sensitive data in descriptions
- ✅ Anomaly detection - ML-based outlier detection
- ✅ Rule versioning - Track which rules applied
- ✅ Canary deployments - Test new rules on 5% traffic
- ✅ Batch processing - Process 1000 transactions at once
- ✅ Redis caching - Cache normalization results for 24h
- ✅ Prometheus metrics - Track model performance
- ✅ Elasticsearch logging - Log all normalizations for audit

**Implementation Complexity:** **HIGH**
- ~3500 lines Python/TypeScript
- scikit-learn for ML model
- Drools rule engine (Java integration)
- Google Places API client
- Multi-currency conversion service
- Redis for caching
- Elasticsearch for logging
- Comprehensive test suite (unit, integration, ML model validation)
- Monitoring dashboards: Model accuracy, API latency, cache hit rate
- A/B testing framework for rule changes

**Narrative Example:**
> Bank processes credit card statement (1,500 transactions). Batch normalization:
> ```python
> observations = observation_store.get_by_upload("UL_12345")  # 1,500 observations
>
> # Batch process for efficiency
> normalizer = MLFinanceNormalizer()
> results = normalizer.normalize_batch(observations, batch_size=1000)
> ```
>
> For each observation:
> 1. **Extract features:**
>    ```python
>    features = {
>        "merchant_text": "STARBUCKS #1234",
>        "amount": -5.75,
>        "category_hint": None,  # No hint from user
>        "time_of_day": 8,  # 8 AM transaction
>        "day_of_week": 1  # Monday
>    }
>    ```
>
> 2. **ML prediction:**
>    ```python
>    ml_result = ml_model.predict(features)
>    # Returns: ("Starbucks", confidence=0.92)
>    ```
>
> 3. **Confidence check:** 0.92 > 0.75 threshold → Use ML result ✅
>
> 4. **External enrichment** (if needed):
>    - Query Google Places API: `"Starbucks #1234"` → `{"name": "Starbucks Coffee", "category": "Cafe", "address": "123 Main St"}`
>    - Cache result in Redis (TTL=30 days)
>
> 5. **Multi-currency** (if transaction in EUR):
>    - Query OpenExchangeRates: `EUR → USD rate = 1.10`
>    - Convert: €5.00 × 1.10 = $5.50 USD
>    - Store both: `original_amount=-5.00, original_currency="EUR", amount_usd=-5.50`
>
> 6. **PII detection:**
>    - Scan description for SSN pattern: `\d{3}-\d{2}-\d{4}` → Not found ✅
>    - Scan for credit card: `\d{16}` → Not found ✅
>
> 7. **Anomaly detection:**
>    - ML model flags: $5.75 is normal for Starbucks (not an outlier)
>
> 8. **Create canonical:**
>    ```python
>    canonical = CanonicalTransaction(
>        date=date(2024, 10, 15),
>        amount=-5.75,
>        merchant="Starbucks Coffee",
>        merchant_id="MER_starbucks_001",
>        category="Food & Drink",
>        external_data={"google_places": {...}}
>    )
>    ```
>
> 9. **Scoring:**
>    ```python
>    confidence = (
>        0.92 * 0.6 +  # ML score (92%) × weight (60%) = 0.552
>        0.95 * 0.2 +  # Fuzzy match (95%) × weight (20%) = 0.19
>        1.0 * 0.1 +   # Rule match (1 rule) × weight (10%) = 0.1
>        1.0 * 0.1     # API match (Google) × weight (10%) = 0.1
>    ) = 0.942
>    ```
>
> 10. **Log to Elasticsearch:**
>     ```json
>     {
>       "timestamp": "2024-10-27T10:00:00Z",
>       "upload_id": "UL_12345",
>       "observation_id": "UL_12345:0",
>       "canonical_id": "CT_12345_0",
>       "ml_model_version": "3.5.0",
>       "ml_confidence": 0.92,
>       "final_confidence": 0.942,
>       "features": {"merchant_text": "STARBUCKS #1234", "amount": -5.75, ...},
>       "applied_rules": ["ml_merchant_model", "google_places_enrichment"],
>       "latency_ms": 45
>     }
>     ```
>
> 11. **Prometheus metrics:**
>     ```
>     normalization_total{model_version="3.5",confidence_bucket="0.9-1.0"} ++
>     normalization_duration_seconds{currency="USD"} 0.045
>     ml_model_used_total{result="success"} ++
>     external_api_calls_total{api="google_places",status="200"} ++
>     ```
>
> Total: 1,500 transactions normalized in 67 seconds (45ms per transaction average, parallelized across 16 workers, with caching).
>
> **Canary deployment test:**
> Bank updates merchant normalization rules (version 3.6). Deploy to 5% of traffic:
> - 95% use rules v3.5
> - 5% use rules v3.6
> - Monitor Prometheus: Compare error rates, confidence scores
> - If v3.6 error rate < 5% threshold → Gradually roll out to 100%

---

**Key Insight:** The same `Normalizer` interface works across all 3 profiles. Personal uses hardcoded rules in Python (150 lines); Small business adds YAML rule engine + fuzzy matching (400 lines); Enterprise uses ML models + external APIs + multi-currency + canary deployments (3500 lines). All transform observations → canonicals with confidence scoring.

---

## Interface Contract

**🔧 P1 Fix Applied:** Added generic types for type safety across domains

```python
from abc import ABC, abstractmethod
from typing import TypeVar, Generic, List
from dataclasses import dataclass

TObservation = TypeVar('TObservation')
TCanonical = TypeVar('TCanonical')

@dataclass
class NormalizationResult(Generic[TCanonical]):
    """
    Container for normalization output with metadata.
    """
    canonical: TCanonical
    confidence: float  # 0.0-1.0
    applied_rules: List[str]  # Rule IDs that were applied
    warnings: List[str]  # Non-fatal validation warnings
    metadata: dict  # Additional context

class Normalizer(ABC, Generic[TObservation, TCanonical]):
    @property
    @abstractmethod
    def version(self) -> str:
        """Semantic version of normalizer (e.g., '1.0.0')"""
        pass

    @abstractmethod
    def normalize(
        self,
        observation: TObservation,
        rules: NormalizationRuleSet
    ) -> NormalizationResult[TCanonical]:
        """
        Transform raw observation into validated canonical.

        Args:
            observation: Raw observation from ObservationStore (type-safe)
            rules: Domain-specific normalization rules

        Returns:
            NormalizationResult containing:
            - canonical: Validated canonical record (type-safe)
            - confidence: Float [0.0, 1.0]
            - applied_rules: List of rule IDs
            - warnings: Non-fatal validation warnings

        Raises:
            ValidationError: If observation data is invalid (fatal)
        """
        pass
```

**Type Safety Example:**

```python
# Finance implementation
class FinanceNormalizer(Normalizer[ObservationTransaction, CanonicalTransaction]):
    def normalize(
        self,
        obs: ObservationTransaction,  # Type-checked at compile time
        rules: NormalizationRuleSet
    ) -> NormalizationResult[CanonicalTransaction]:  # Return type enforced
        canonical = CanonicalTransaction(
            canonical_id=generate_id(obs.upload_id, obs.row_id),
            date=parse_iso_date(obs.raw_data['date']),
            amount=Decimal(obs.raw_data['amount']),
            ...
        )
        return NormalizationResult(
            canonical=canonical,
            confidence=0.95,
            applied_rules=['date_iso', 'amount_decimal'],
            warnings=[],
            metadata={}
        )

# Healthcare implementation
class HealthcareNormalizer(Normalizer[ObservationLabResult, CanonicalLabResult]):
    def normalize(
        self,
        obs: ObservationLabResult,  # Different type, same interface
        rules: NormalizationRuleSet
    ) -> NormalizationResult[CanonicalLabResult]:
        canonical = CanonicalLabResult(
            result_id=generate_id(obs.upload_id, obs.row_id),
            test_date=parse_iso_date(obs.raw_data['date']),
            value=Decimal(obs.raw_data['value']),
            unit=obs.raw_data['unit'],
            ...
        )
        return NormalizationResult(
            canonical=canonical,
            confidence=0.98,
            applied_rules=['date_iso', 'value_decimal', 'unit_loinc'],
            warnings=[],
            metadata={}
        )
```

**Benefits of Generic Types:**
- ✅ Compile-time type checking (catch errors before runtime)
- ✅ Better IDE autocomplete (knows canonical.date exists)
- ✅ Self-documenting code (clear input/output types)
- ✅ Prevents mixing domains (can't pass ObservationTransaction to HealthcareNormalizer)
```

---

## Behavior

**Transformation example (Finance):**
```python
# Input: Raw observation
observation = ObservationTransaction(
    upload_id="UL_abc123",
    row_id=0,
    raw_data={
        "date": "01/15/2024",        # String (ambiguous format)
        "amount": "-5.75",            # String
        "description": "  STARBUCKS #1234  "  # Dirty
    }
)

# Output: Canonical transaction
canonical = normalizer.normalize(observation, rules)
# canonical.date = "2025-01-15T00:00:00Z" (ISO 8601)
# canonical.amount = -5.75 (Decimal)
# canonical.description = "Starbucks #1234" (cleaned)
# canonical.merchant = "Starbucks" (extracted)
# canonical.category = "Food & Drink" (inferred)
```

---

## Responsibilities

✅ **DOES:**
- Validate field formats (dates, amounts, currencies)
- Transform raw strings → typed values (ISO dates, decimals)
- Clean text (trim spaces, normalize case)
- Extract structured data (merchant from description)
- Infer metadata (category from merchant)
- Apply business rules (duplicate detection)
- Generate confidence scores

❌ **DOES NOT:**
- Modify raw observations (they remain immutable)
- Update UploadRecord.status (Coordinator only)
- Make multi-transaction decisions (transfers → vertical 3.5 or 3.9)

---

## Multi-Domain Applicability

**Finance:** ObservationTransaction → CanonicalTransaction (dates, amounts, categories)
**Healthcare:** ObservationLabResult → CanonicalLabResult (numeric values, units, ranges)
**Legal:** ObservationClause → CanonicalClause (clause types, obligations, dates)
**Research (RSRCH - Utilitario):** RawFact → CanonicalFact (entity names, fact claims, source URLs)
**Manufacturing:** ObservationMeasurement → CanonicalMeasurement (sensor values, units)
**Media:** ObservationUtterance → CanonicalUtterance (timestamps, speaker IDs)

---

## Domain Validation

### ✅ Finance (Primary Instantiation)
**Use case:** Transform raw bank CSV row into validated canonical transaction
**Example:** Raw observation `{"date": "01/15/2024", "amount": "-5.75", "description": "  STARBUCKS #1234  "}` → Canonical with ISO date "2025-01-15T00:00:00Z", Decimal amount -5.75, merchant "Starbucks", category "Food & Drink"
**Fields validated:** `date` (format: MM/DD/YYYY → ISO 8601), `amount` (string → Decimal), `merchant` (extracted from description), `category` (inferred)
**Rules applied:** date_iso, amount_decimal, merchant_extract, category_infer
**Status:** ✅ Fully implemented in personal-finance-app

### ✅ Healthcare
**Use case:** Transform raw lab result text into validated canonical lab result
**Example:** Raw observation `{"date": "2024-03-15", "test": "Glucose", "value": "95", "unit": "mg/dL"}` → Canonical with normalized test code (LOINC 2345-7), validated range (normal: 70-100), flagged abnormal results
**Fields validated:** `test_date` (ISO 8601), `value` (string → Decimal), `unit` (standardized to UCUM), `reference_range` (age/gender-specific)
**Rules applied:** date_iso, value_decimal, unit_ucum, range_validate, loinc_map
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ Legal
**Use case:** Transform raw contract clause text into structured canonical clause
**Example:** Raw observation `{"clause": "Payment due within 30 days of invoice date", "section": "3.2"}` → Canonical with clause_type "payment_terms", obligation_type "payment", deadline_days 30, enforceability "mandatory"
**Fields validated:** `clause_type` (classified), `obligation_type` (extracted), `deadline_days` (parsed from text), `enforceability` (inferred)
**Rules applied:** clause_classify, obligation_extract, deadline_parse, enforceability_infer
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ RSRCH (Utilitario Research)
**Use case:** Transform raw founder mention into canonical entity record
**Example:** Raw observation `{"text": "@sama invested $375M in OpenAI", "source": "techcrunch.com", "date": "2024-02-25"}` → Canonical with entity_name "Sam Altman", entity_type "person", company "OpenAI", investment_amount 375000000 (parsed), source_type "news"
**Fields validated:** `entity_name` (normalized "@sama" → "Sam Altman"), `investment_amount` (string "$375M" → Decimal 375000000), `source_type` (classified), `fact_date` (ISO 8601)
**Rules applied:** entity_normalize, amount_parse, source_classify, date_iso
**Status:** ✅ Conceptually validated via examples in this doc

### ✅ E-commerce
**Use case:** Transform raw product scrape data into validated canonical product record
**Example:** Raw observation `{"title": "  iPhone 15 Pro Max - 256GB  ", "price": "$1,199.99", "category": "Electronics > Phones"}` → Canonical with clean title "iPhone 15 Pro Max 256GB", price Decimal(1199.99), category hierarchy ["Electronics", "Phones"], SKU generated
**Fields validated:** `title` (cleaned, normalized), `price` (string → Decimal), `category` (hierarchy parsed), `SKU` (generated), `availability` (inferred)
**Rules applied:** text_clean, price_decimal, category_parse, sku_generate
**Status:** ✅ Conceptually validated via examples in this doc

**Validation Status:** ✅ **5 domains validated** (1 fully implemented, 4 conceptually verified)
**Domain-Agnostic Score:** 100% (generic interface with TypeVar[TObservation, TCanonical])
**Reusability:** High (same normalize() method works across all domains, only validation rules differ)

---

## Related Primitives

- **ValidationEngine**: Used by Normalizer for field-level validation
- **NormalizationRuleSet**: Configuration for domain-specific rules
- **ObservationStore**: Source of raw observations
- **CanonicalStore**: Destination for normalized canonicals
- **NormalizationLog**: Records normalization execution details

---

**Last Updated**: 2025-10-27
**Maturity**: Spec complete, ready for implementation
