# TaxCategoryClassifier (OL Primitive)

**Vertical:** 3.4 Tax Categorization (Regulatory Classification)
**Layer:** Objective Layer (OL)
**Type:** Pattern Matching + ML Classification Engine
**Status:** ✅ Specified

---

## Purpose

Auto-suggest tax categories based on transaction patterns using rule-based matching and ML-based prediction. Provides confidence scores, historical pattern analysis, and batch optimization for performance.

**Core Responsibility:** Analyze transaction attributes (merchant, amount, date patterns, user history) to suggest appropriate tax categories with confidence scoring. Support both rule-based matching (deterministic) and ML-based prediction (probabilistic) to minimize manual categorization effort.

---

## Multi-Domain Applicability

This primitive is **domain-agnostic**. Same pattern applies to ANY classification/categorization task:

| Domain | Classification Task | Input Features | Suggested Categories |
|--------|---------------------|----------------|----------------------|
| **Finance** | Tax category | Merchant name, amount, account, historical patterns | Schedule C categories, SAT categories |
| **Healthcare** | Procedure code | Provider name, diagnosis, treatment type, insurance | CPT codes, ICD-10 codes |
| **Legal** | Case category | Client name, case description, jurisdiction, attorney | Case law categories, filing types |
| **Research** | Grant expense category | Vendor name, expense type, amount, grant | NSF categories, NIH allowable costs |
| **E-commerce** | Product category | Product title, description, brand, price | Product taxonomy, department |
| **Content** | Content tags | Article text, author, publication, topic | Content categories, tags |

**Pattern:** Auto-Classification = analyze input features → match historical patterns (rule-based) → predict using ML model (probabilistic) → return suggestions with confidence scores → learn from user feedback.

---

## Interface Contract

### Python Interface

```python
from typing import Optional, List, Dict
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal

@dataclass
class CategorySuggestion:
    """Represents a suggested category with confidence score."""
    category_id: str
    category_name: str
    confidence: float  # [0.0, 1.0]
    reason: str  # Human-readable explanation
    method: str  # "rule_based", "ml_based", "historical_pattern"
    metadata: Dict  # Additional info (historical_count, pattern_details, etc.)

@dataclass
class ClassificationResult:
    """Result of classification operation."""
    transaction_id: str
    suggestions: List[CategorySuggestion]  # Ordered by confidence DESC
    top_suggestion: Optional[CategorySuggestion]  # Highest confidence (if >= threshold)
    classified_at: datetime

class TaxCategoryClassifier:
    """
    Auto-suggest tax categories based on transaction patterns.

    Classification strategies (in order):
    1. Exact merchant match (confidence: 0.95+)
    2. Fuzzy merchant match (confidence: 0.70-0.95)
    3. Amount pattern match (confidence: 0.60-0.80)
    4. ML model prediction (confidence: 0.50-0.90)
    5. No match (confidence: < 0.50)

    Minimum confidence threshold: 0.70 (configurable)
    """

    def __init__(
        self,
        category_store,
        counterparty_store,
        transaction_store,
        min_confidence: float = 0.70
    ):
        self.category_store = category_store
        self.counterparty_store = counterparty_store
        self.transaction_store = transaction_store
        self.min_confidence = min_confidence
        self._model_cache = {}  # Cache ML models per user

    def auto_classify(
        self,
        transaction: Transaction,
        user_id: str,
        jurisdiction: str,
        max_suggestions: int = 3
    ) -> ClassificationResult:
        """
        Auto-suggest tax category for transaction.

        Classification process:
        1. Extract features (merchant, amount, date, account)
        2. Check historical patterns (exact merchant match)
        3. Check fuzzy patterns (similar merchants)
        4. Check amount patterns (similar amounts)
        5. Use ML model (if trained)
        6. Rank by confidence, return top N

        Args:
            transaction: Transaction to classify
            user_id: Owner of transaction
            jurisdiction: Regulatory framework (usa_federal, mexico_federal, etc.)
            max_suggestions: Maximum suggestions to return (default: 3)

        Returns:
            ClassificationResult with suggestions ordered by confidence

        Example:
            # Classify Uber transaction
            result = classifier.auto_classify(
                transaction=uber_txn,
                user_id="user_darwin",
                jurisdiction="usa_federal"
            )

            if result.top_suggestion:
                print(f"Suggested: {result.top_suggestion.category_name}")
                print(f"Confidence: {result.top_suggestion.confidence:.0%}")
                print(f"Reason: {result.top_suggestion.reason}")
            # Output:
            # Suggested: Travel
            # Confidence: 95%
            # Reason: Based on 10 similar Uber transactions
        """

    def bulk_classify(
        self,
        transactions: List[Transaction],
        user_id: str,
        jurisdiction: str
    ) -> Dict[str, ClassificationResult]:
        """
        Classify multiple transactions in batch (optimized).

        Batch optimization:
        - Load all historical patterns once
        - Load ML model once
        - Reuse counterparty lookups

        Args:
            transactions: List of transactions to classify
            user_id: Owner of transactions
            jurisdiction: Regulatory framework

        Returns:
            Dictionary: transaction_id → ClassificationResult

        Example:
            # Classify 100 uncategorized transactions
            uncategorized = transaction_store.list(
                user_id="user_darwin",
                has_tax_classification=False,
                limit=100
            )

            results = classifier.bulk_classify(
                transactions=uncategorized,
                user_id="user_darwin",
                jurisdiction="usa_federal"
            )

            # Apply auto-suggestions with high confidence
            for txn_id, result in results.items():
                if result.top_suggestion and result.top_suggestion.confidence >= 0.90:
                    # Auto-apply
                    transaction_store.classify(txn_id, result.top_suggestion.category_id)
        """

    def train_from_history(
        self,
        user_id: str,
        jurisdiction: str,
        min_samples: int = 50
    ) -> Dict:
        """
        Train ML classifier from user's classification history.

        Training data:
        - Features: merchant_name, amount_range, day_of_week, account_id
        - Labels: category_id
        - Minimum samples: 50 (configurable)

        ML model: Naive Bayes or Logistic Regression (lightweight)

        Args:
            user_id: Owner of training data
            jurisdiction: Regulatory framework
            min_samples: Minimum classified transactions required (default: 50)

        Returns:
            Training metrics: {
                "samples_count": 150,
                "categories_count": 8,
                "accuracy": 0.85,
                "trained_at": "2025-05-15T10:00:00Z"
            }

        Raises:
            InsufficientDataError: If fewer than min_samples classified transactions

        Example:
            # Train after user has classified 100+ transactions
            metrics = classifier.train_from_history(
                user_id="user_darwin",
                jurisdiction="usa_federal"
            )

            print(f"Trained on {metrics['samples_count']} transactions")
            print(f"Model accuracy: {metrics['accuracy']:.0%}")
        """

    def explain_suggestion(
        self,
        transaction: Transaction,
        suggestion: CategorySuggestion
    ) -> Dict:
        """
        Explain why category was suggested (interpretability).

        Returns detailed breakdown of features that influenced suggestion.

        Args:
            transaction: Transaction being classified
            suggestion: Suggestion to explain

        Returns:
            Explanation dictionary: {
                "method": "historical_pattern",
                "features": {
                    "merchant_name": {"value": "Uber", "weight": 0.8},
                    "amount_range": {"value": "$20-$40", "weight": 0.1},
                    "day_of_week": {"value": "weekday", "weight": 0.1}
                },
                "historical_matches": 12,
                "historical_accuracy": 0.95
            }

        Example:
            explanation = classifier.explain_suggestion(txn, suggestion)
            print(f"Method: {explanation['method']}")
            print(f"Top feature: {explanation['features']['merchant_name']['value']}")
        """

    def get_classification_patterns(
        self,
        user_id: str,
        jurisdiction: str
    ) -> List[Dict]:
        """
        Get user's classification patterns (merchant → category mapping).

        Useful for debugging and understanding user's categorization habits.

        Args:
            user_id: Owner of patterns
            jurisdiction: Regulatory framework

        Returns:
            List of patterns: [
                {
                    "merchant_name": "Uber",
                    "category_id": "tax_cat_usa_schedule_c_travel",
                    "category_name": "Travel",
                    "count": 15,
                    "avg_amount": -25.50,
                    "last_classified_at": "2025-05-14T12:00:00Z"
                }
            ]

        Example:
            patterns = classifier.get_classification_patterns(
                user_id="user_darwin",
                jurisdiction="usa_federal"
            )

            for p in patterns:
                print(f"{p['merchant_name']} → {p['category_name']} ({p['count']}x)")
        """

    def _match_exact_merchant(
        self,
        user_id: str,
        jurisdiction: str,
        counterparty_id: str
    ) -> Optional[CategorySuggestion]:
        """
        Check for exact merchant match in historical patterns.

        Strategy:
        - Query transactions with same counterparty_id
        - Group by category_id
        - Return most frequent category with confidence

        Confidence calculation:
        - 100% agreement (all same category) → 0.95
        - 80%+ agreement → 0.85
        - 60%+ agreement → 0.75
        - < 60% agreement → None (too ambiguous)

        Internal method.
        """

    def _match_fuzzy_merchant(
        self,
        user_id: str,
        jurisdiction: str,
        merchant_name: str
    ) -> Optional[CategorySuggestion]:
        """
        Check for fuzzy merchant match (similar names).

        Strategy:
        - Find counterparties with similar names (Levenshtein distance)
        - Check their category assignments
        - Return if strong pattern exists

        Confidence: 0.70-0.85 (lower than exact match)

        Internal method.
        """

    def _match_amount_pattern(
        self,
        user_id: str,
        jurisdiction: str,
        amount: Decimal,
        tolerance: Decimal = Decimal("10.0")
    ) -> Optional[CategorySuggestion]:
        """
        Check for amount pattern match.

        Strategy:
        - Find transactions with similar amounts (±tolerance)
        - Group by category_id
        - Return most frequent category

        Confidence: 0.60-0.80 (less reliable than merchant)

        Internal method.
        """

    def _predict_ml(
        self,
        user_id: str,
        jurisdiction: str,
        transaction: Transaction
    ) -> Optional[CategorySuggestion]:
        """
        Use ML model to predict category.

        Features:
        - merchant_name (hashed)
        - amount_range (bucketed: <$10, $10-$50, $50-$100, >$100)
        - day_of_week
        - account_id (hashed)

        Model: Naive Bayes or Logistic Regression

        Confidence: Model probability (0.50-0.90)

        Internal method.
        """
```

---

## Implementation Details

### Exact Merchant Matching

```python
def _match_exact_merchant(
    self,
    user_id: str,
    jurisdiction: str,
    counterparty_id: str
) -> Optional[CategorySuggestion]:
    """
    Match based on exact counterparty history.

    Algorithm:
    1. Query all transactions with this counterparty
    2. Filter by has tax_classification
    3. Group by category_id
    4. Calculate confidence based on consistency

    Confidence formula:
      confidence = (most_frequent_count / total_count) * 0.95
      (Cap at 0.95 to distinguish from manual classification)
    """
    # Query historical transactions
    historical = self.transaction_store.list(
        user_id=user_id,
        counterparty_id=counterparty_id,
        has_tax_classification=True
    )

    if not historical:
        return None  # No history

    # Group by category
    category_counts = {}
    for txn in historical:
        cat_id = txn.tax_classification.get('category_id')
        if cat_id:
            category_counts[cat_id] = category_counts.get(cat_id, 0) + 1

    # Find most frequent category
    most_frequent_cat = max(category_counts, key=category_counts.get)
    count = category_counts[most_frequent_cat]
    total = len(historical)

    # Calculate confidence
    agreement_ratio = count / total
    if agreement_ratio < 0.60:
        return None  # Too ambiguous

    confidence = min(agreement_ratio * 0.95, 0.95)

    # Get category details
    category = self.category_store.get(most_frequent_cat)

    return CategorySuggestion(
        category_id=category.category_id,
        category_name=category.name,
        confidence=confidence,
        reason=f"Based on {count} of {total} similar transactions with this merchant",
        method="exact_merchant_match",
        metadata={
            'historical_count': count,
            'total_count': total,
            'agreement_ratio': agreement_ratio
        }
    )
```

### Fuzzy Merchant Matching

```python
def _match_fuzzy_merchant(
    self,
    user_id: str,
    jurisdiction: str,
    merchant_name: str
) -> Optional[CategorySuggestion]:
    """
    Match based on similar merchant names.

    Algorithm:
    1. Find counterparties with similar names (Levenshtein distance < 3)
    2. Get their category assignments
    3. Return if strong pattern exists

    Confidence: Lower than exact match (0.70-0.85)
    """
    import Levenshtein  # pip install python-Levenshtein

    # Get all counterparties for user
    counterparties = self.counterparty_store.list(user_id=user_id)

    # Find similar names
    similar_counterparties = []
    for cp in counterparties:
        distance = Levenshtein.distance(
            merchant_name.lower(),
            cp.canonical_name.lower()
        )

        if distance <= 3:  # Close match
            similar_counterparties.append(cp)

    if not similar_counterparties:
        return None

    # Get category assignments for similar counterparties
    category_counts = {}
    total_txns = 0

    for cp in similar_counterparties:
        historical = self.transaction_store.list(
            user_id=user_id,
            counterparty_id=cp.counterparty_id,
            has_tax_classification=True
        )

        for txn in historical:
            cat_id = txn.tax_classification.get('category_id')
            if cat_id:
                category_counts[cat_id] = category_counts.get(cat_id, 0) + 1
                total_txns += 1

    if not category_counts:
        return None

    # Find most frequent
    most_frequent_cat = max(category_counts, key=category_counts.get)
    count = category_counts[most_frequent_cat]

    agreement_ratio = count / total_txns
    if agreement_ratio < 0.70:
        return None

    # Lower confidence than exact match
    confidence = min(agreement_ratio * 0.85, 0.85)

    category = self.category_store.get(most_frequent_cat)

    return CategorySuggestion(
        category_id=category.category_id,
        category_name=category.name,
        confidence=confidence,
        reason=f"Based on {count} similar transactions with similar merchants",
        method="fuzzy_merchant_match",
        metadata={
            'similar_merchants': [cp.canonical_name for cp in similar_counterparties],
            'historical_count': count,
            'total_count': total_txns
        }
    )
```

### Amount Pattern Matching

```python
def _match_amount_pattern(
    self,
    user_id: str,
    jurisdiction: str,
    amount: Decimal,
    tolerance: Decimal = Decimal("10.0")
) -> Optional[CategorySuggestion]:
    """
    Match based on similar transaction amounts.

    Algorithm:
    1. Find transactions with amount in range [amount - tolerance, amount + tolerance]
    2. Filter by has tax_classification
    3. Group by category
    4. Return most frequent

    Confidence: 0.60-0.80 (less reliable than merchant)
    """
    # Query transactions in amount range
    historical = self.transaction_store.list(
        user_id=user_id,
        amount_min=amount - tolerance,
        amount_max=amount + tolerance,
        has_tax_classification=True
    )

    if len(historical) < 5:
        return None  # Too few samples

    # Group by category
    category_counts = {}
    for txn in historical:
        cat_id = txn.tax_classification.get('category_id')
        if cat_id:
            category_counts[cat_id] = category_counts.get(cat_id, 0) + 1

    if not category_counts:
        return None

    # Find most frequent
    most_frequent_cat = max(category_counts, key=category_counts.get)
    count = category_counts[most_frequent_cat]
    total = len(historical)

    agreement_ratio = count / total
    if agreement_ratio < 0.60:
        return None

    # Lower confidence
    confidence = min(agreement_ratio * 0.80, 0.80)

    category = self.category_store.get(most_frequent_cat)

    return CategorySuggestion(
        category_id=category.category_id,
        category_name=category.name,
        confidence=confidence,
        reason=f"Based on {count} transactions with similar amounts (${amount} ±${tolerance})",
        method="amount_pattern_match",
        metadata={
            'amount': float(amount),
            'tolerance': float(tolerance),
            'historical_count': count,
            'total_count': total
        }
    )
```

### ML-Based Prediction

```python
def _predict_ml(
    self,
    user_id: str,
    jurisdiction: str,
    transaction: Transaction
) -> Optional[CategorySuggestion]:
    """
    Use ML model to predict category.

    Model: Naive Bayes (lightweight, interpretable)

    Features:
    1. merchant_name (hashed to integer)
    2. amount_bucket (categorical: <10, 10-50, 50-100, >100)
    3. day_of_week (0-6)
    4. account_id (hashed to integer)

    Training: train_from_history()
    """
    # Check if model exists for user
    model_key = f"{user_id}:{jurisdiction}"
    if model_key not in self._model_cache:
        return None  # No model trained

    model = self._model_cache[model_key]

    # Extract features
    features = self._extract_features(transaction)

    # Predict
    try:
        probabilities = model.predict_proba([features])[0]
        predicted_class = model.classes_[probabilities.argmax()]
        confidence = float(probabilities.max())

        if confidence < 0.50:
            return None  # Too uncertain

        category = self.category_store.get(predicted_class)

        return CategorySuggestion(
            category_id=category.category_id,
            category_name=category.name,
            confidence=confidence,
            reason=f"ML model prediction based on transaction patterns",
            method="ml_prediction",
            metadata={
                'model_confidence': confidence,
                'model_type': 'naive_bayes'
            }
        )
    except Exception as e:
        # Log error, return None
        return None

def _extract_features(self, transaction: Transaction) -> List:
    """
    Extract features for ML model.

    Returns: [merchant_hash, amount_bucket, day_of_week, account_hash]
    """
    # Hash merchant name
    merchant_hash = hash(transaction.counterparty_id) % 10000

    # Bucket amount
    amount = abs(transaction.amount)
    if amount < 10:
        amount_bucket = 0
    elif amount < 50:
        amount_bucket = 1
    elif amount < 100:
        amount_bucket = 2
    else:
        amount_bucket = 3

    # Day of week
    day_of_week = transaction.date.weekday()

    # Hash account
    account_hash = hash(transaction.account_id) % 1000

    return [merchant_hash, amount_bucket, day_of_week, account_hash]

def train_from_history(
    self,
    user_id: str,
    jurisdiction: str,
    min_samples: int = 50
) -> Dict:
    """
    Train Naive Bayes classifier from user's history.

    Training process:
    1. Fetch all classified transactions
    2. Extract features (merchant, amount_bucket, day_of_week, account)
    3. Train Naive Bayes model
    4. Evaluate with cross-validation
    5. Cache model
    """
    from sklearn.naive_bayes import MultinomialNB
    from sklearn.model_selection import cross_val_score

    # Fetch training data
    historical = self.transaction_store.list(
        user_id=user_id,
        has_tax_classification=True,
        limit=1000  # Cap at 1000
    )

    if len(historical) < min_samples:
        raise InsufficientDataError(
            f"Need at least {min_samples} classified transactions, found {len(historical)}"
        )

    # Extract features and labels
    X = []
    y = []

    for txn in historical:
        features = self._extract_features(txn)
        category_id = txn.tax_classification.get('category_id')

        X.append(features)
        y.append(category_id)

    # Train model
    model = MultinomialNB()
    model.fit(X, y)

    # Evaluate
    scores = cross_val_score(model, X, y, cv=5)
    accuracy = scores.mean()

    # Cache model
    model_key = f"{user_id}:{jurisdiction}"
    self._model_cache[model_key] = model

    return {
        'samples_count': len(historical),
        'categories_count': len(set(y)),
        'accuracy': accuracy,
        'trained_at': datetime.utcnow().isoformat()
    }
```

---

## Usage Examples

### Example 1: Auto-Classify Single Transaction

```python
classifier = TaxCategoryClassifier(
    category_store=category_store,
    counterparty_store=counterparty_store,
    transaction_store=transaction_store
)

# Classify Uber transaction
uber_txn = transaction_store.get("txn_12345")

result = classifier.auto_classify(
    transaction=uber_txn,
    user_id="user_darwin",
    jurisdiction="usa_federal"
)

if result.top_suggestion:
    print(f"Suggested: {result.top_suggestion.category_name}")
    print(f"Confidence: {result.top_suggestion.confidence:.0%}")
    print(f"Reason: {result.top_suggestion.reason}")
else:
    print("No confident suggestion found")

# Output:
# Suggested: Travel
# Confidence: 95%
# Reason: Based on 10 similar transactions with this merchant
```

### Example 2: Bulk Classify Uncategorized Transactions

```python
# Get uncategorized transactions
uncategorized = transaction_store.list(
    user_id="user_darwin",
    has_tax_classification=False,
    limit=100
)

# Bulk classify
results = classifier.bulk_classify(
    transactions=uncategorized,
    user_id="user_darwin",
    jurisdiction="usa_federal"
)

# Auto-apply high-confidence suggestions
auto_applied = 0
for txn_id, result in results.items():
    if result.top_suggestion and result.top_suggestion.confidence >= 0.90:
        transaction_store.classify(
            transaction_id=txn_id,
            category_id=result.top_suggestion.category_id,
            classification_method="auto_applied"
        )
        auto_applied += 1

print(f"Auto-applied {auto_applied} of {len(results)} suggestions")
```

### Example 3: Train ML Model

```python
# Train after user has classified enough transactions
try:
    metrics = classifier.train_from_history(
        user_id="user_darwin",
        jurisdiction="usa_federal"
    )

    print(f"✓ Model trained successfully")
    print(f"  Samples: {metrics['samples_count']}")
    print(f"  Categories: {metrics['categories_count']}")
    print(f"  Accuracy: {metrics['accuracy']:.0%}")
except InsufficientDataError as e:
    print(f"✗ {e.message}")
```

### Example 4: Explain Suggestion

```python
result = classifier.auto_classify(uber_txn, "user_darwin", "usa_federal")

if result.top_suggestion:
    explanation = classifier.explain_suggestion(uber_txn, result.top_suggestion)

    print(f"Method: {explanation['method']}")
    print(f"Features:")
    for feature_name, feature_data in explanation['features'].items():
        print(f"  {feature_name}: {feature_data['value']} (weight: {feature_data['weight']})")
```

### Example 5: View Classification Patterns

```python
patterns = classifier.get_classification_patterns(
    user_id="user_darwin",
    jurisdiction="usa_federal"
)

print("Top classification patterns:")
for p in patterns[:10]:
    print(f"{p['merchant_name']:30} → {p['category_name']:20} ({p['count']:3}x, avg: ${p['avg_amount']:7.2f})")

# Output:
# Uber                           → Travel               ( 15x, avg: $ -25.50)
# Starbucks                      → Meals                ( 22x, avg: $  -6.75)
# Amazon                         → Office Expenses      ( 48x, avg: $ -35.20)
```

---

## Multi-Domain Examples

### Healthcare: Auto-Classify Medical Claims

```python
# Classify medical claim
claim = claim_store.get("claim_12345")

result = classifier.auto_classify(
    transaction=claim,  # Reuse same interface
    user_id="doctor_123",
    jurisdiction="cpt"
)

# Suggested CPT code based on diagnosis and provider history
# "Office Visit - Established Patient (99213)" with 85% confidence
```

### Legal: Auto-Classify Case Expenses

```python
# Classify legal expense
expense = expense_store.get("exp_67890")

result = classifier.auto_classify(
    transaction=expense,
    user_id="lawyer_456",
    jurisdiction="federal"
)

# Suggested case category based on vendor and amount patterns
# "Discovery - Document Review" with 78% confidence
```

### Research: Auto-Classify Grant Expenses

```python
# Classify research expense
expense = expense_store.get("exp_nsf_001")

result = classifier.auto_classify(
    transaction=expense,
    user_id="researcher_789",
    jurisdiction="nsf"
)

# Suggested NSF category based on vendor and historical patterns
# "Personnel - Graduate Students" with 92% confidence
```

---

## Confidence Scoring

### Confidence Ranges

| Method | Confidence Range | Reliability |
|--------|------------------|-------------|
| Exact merchant match (100% agreement) | 0.95 | Very High |
| Exact merchant match (80%+ agreement) | 0.85 | High |
| Fuzzy merchant match | 0.70-0.85 | Medium-High |
| Amount pattern match | 0.60-0.80 | Medium |
| ML prediction (high probability) | 0.70-0.90 | Medium-High |
| ML prediction (low probability) | 0.50-0.70 | Low |

### Threshold Recommendations

| Confidence | Action | Use Case |
|-----------|--------|----------|
| ≥ 0.95 | Auto-apply | Very confident, minimal risk |
| 0.85-0.94 | Auto-suggest (default accept) | High confidence, user can reject |
| 0.70-0.84 | Show as option | Medium confidence, user decides |
| 0.50-0.69 | Show as weak suggestion | Low confidence, informational only |
| < 0.50 | Don't show | Too uncertain |

---

## Performance Characteristics

### Query Performance

| Operation | Expected Latency (p95) | Optimization |
|-----------|------------------------|--------------|
| `auto_classify()` | <200ms | Cache historical patterns |
| `bulk_classify()` (100 txns) | <2000ms | Batch queries, reuse patterns |
| `train_from_history()` | <5000ms | Run async, cache model |

### Batch Optimization

```python
def bulk_classify(
    self,
    transactions: List[Transaction],
    user_id: str,
    jurisdiction: str
) -> Dict[str, ClassificationResult]:
    """
    Optimize for batch processing.

    Optimizations:
    1. Load all historical patterns once (not per transaction)
    2. Load ML model once (if exists)
    3. Batch counterparty lookups
    4. Cache intermediate results
    """
    # Pre-load historical patterns
    all_historical = self.transaction_store.list(
        user_id=user_id,
        has_tax_classification=True,
        limit=5000
    )

    # Build pattern maps
    merchant_patterns = self._build_merchant_patterns(all_historical)
    amount_patterns = self._build_amount_patterns(all_historical)

    # Load ML model (if exists)
    model_key = f"{user_id}:{jurisdiction}"
    ml_model = self._model_cache.get(model_key)

    # Classify each transaction using cached data
    results = {}
    for txn in transactions:
        result = self._classify_with_cache(
            txn,
            merchant_patterns,
            amount_patterns,
            ml_model
        )
        results[txn.transaction_id] = result

    return results
```

---

## Error Handling

### Custom Exceptions

```python
class ClassifierError(Exception):
    """Base exception for classifier errors."""
    pass

class InsufficientDataError(ClassifierError):
    def __init__(self, message: str, samples_found: int = 0, samples_required: int = 50):
        self.message = message
        self.samples_found = samples_found
        self.samples_required = samples_required
        super().__init__(message)

class ModelNotTrainedError(ClassifierError):
    pass

class InvalidFeatureError(ClassifierError):
    pass
```

---

## Testing Guidance

### Unit Test Coverage

```python
# Test: Exact merchant match
def test_exact_merchant_match_100_percent_agreement()
def test_exact_merchant_match_80_percent_agreement()
def test_exact_merchant_match_no_history()

# Test: Fuzzy merchant match
def test_fuzzy_merchant_match_similar_names()
def test_fuzzy_merchant_match_no_similar()

# Test: Amount pattern match
def test_amount_pattern_match_in_range()
def test_amount_pattern_match_insufficient_samples()

# Test: ML prediction
def test_ml_predict_high_confidence()
def test_ml_predict_low_confidence()
def test_ml_predict_no_model()

# Test: Auto-classify
def test_auto_classify_exact_match()
def test_auto_classify_multiple_suggestions()
def test_auto_classify_no_suggestions()

# Test: Bulk classify
def test_bulk_classify_performance()
def test_bulk_classify_reuse_patterns()

# Test: Train model
def test_train_from_history_success()
def test_train_from_history_insufficient_data()
def test_train_from_history_cache_model()
```

---

## Related Primitives

- **TaxCategoryStore** (OL) - Provides category data
- **CounterpartyStore** (OL) - Provides merchant data
- **TransactionStore** (OL) - Provides transaction history
- **TaxonomyEngine** (OL) - Provides category hierarchy for path-based matching
- **TaxCategorySelector** (IL) - Displays suggestions in UI

---

**Status:** ✅ Ready for implementation
**Estimated Implementation Time:** 4-5 days
**Dependencies:** TaxCategoryStore, CounterpartyStore, TransactionStore, scikit-learn (ML), python-Levenshtein (fuzzy matching)
