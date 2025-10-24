# Primitive: ClusteringEngine (OL)

> **Layer:** Objective Layer (OL)
> **Type:** Clustering & Similarity Analysis
> **Vertical:** 3.8 Cluster Rules
> **Last Updated:** 2025-10-24

---

## Overview

**ClusteringEngine** is an Objective Layer primitive that clusters similar transactions, entities, or text strings using density-based or hierarchical clustering algorithms. It auto-detects entity variants and suggests merges for normalization.

**Core Responsibility:**
Group similar items (merchants, providers, institutions, etc.) based on string similarity, enabling batch normalization and entity deduplication.

**Domain Instantiation:**
- Finance: Cluster merchant name variants ("AMAZON.COM", "AMZN MKTP US" → "Amazon")
- Healthcare: Cluster provider name variations ("ST MARY'S", "ST MARY HOSP" → "St. Mary's Hospital")
- Legal: Cluster related case names for matter grouping
- Research: Cluster institution/author name variants for citation deduplication

---

## Interface

### Methods

```python
class ClusteringEngine:
    """
    Clusters similar items using density-based or hierarchical algorithms.

    Attributes:
        fuzzy_matcher: FuzzyMatcher instance for similarity calculation
        min_similarity: Default similarity threshold [0.0, 1.0]
        min_cluster_size: Minimum items per cluster
    """

    def create_clusters(self, items: List[str], similarity_threshold: float = 0.70,
                        algorithm: str = "dbscan",
                        min_cluster_size: int = 2) -> List[Cluster]:
        """
        Clusters similar items using specified algorithm.

        Algorithms:
        - dbscan: Density-based clustering (default, handles noise well)
        - hierarchical: Agglomerative clustering (builds tree)
        - simple: Threshold-based grouping (fast but less accurate)

        Args:
            items: List of strings to cluster (e.g., merchant names)
            similarity_threshold: [0.0, 1.0], min similarity to group (default 0.70)
            algorithm: "dbscan" | "hierarchical" | "simple"
            min_cluster_size: Minimum items per cluster (default 2)

        Returns:
            List of Cluster objects with canonical name and variants

        Example:
            >>> items = ["AMAZON.COM", "AMZN MKTP US", "AMAZON PRIME", "WALMART"]
            >>> clusters = engine.create_clusters(items, similarity_threshold=0.70)
            >>> len(clusters)
            2  # Amazon cluster + Walmart cluster (outlier)
            >>> clusters[0].canonical_name
            "Amazon"
            >>> len(clusters[0].variants)
            3  # AMAZON.COM, AMZN MKTP US, AMAZON PRIME
        """

    def calculate_similarity_matrix(self, items: List[str],
                                     algorithm: str = "jaro_winkler") -> np.ndarray:
        """
        Calculates pairwise similarity matrix for clustering.

        Args:
            items: List of strings
            algorithm: "levenshtein" | "jaro_winkler" | "soundex" | "metaphone"

        Returns:
            NxN numpy array where [i][j] = similarity(items[i], items[j])

        Example:
            >>> items = ["AMAZON", "AMZN", "WALMART"]
            >>> matrix = engine.calculate_similarity_matrix(items)
            >>> matrix[0][1]  # AMAZON vs AMZN
            0.78
            >>> matrix[0][2]  # AMAZON vs WALMART
            0.15
        """

    def suggest_merge(self, cluster_id: str, user_id: str) -> MergeSuggestion:
        """
        Suggests normalization rules for cluster.

        Args:
            cluster_id: Cluster ID to process
            user_id: User ID for rule creation

        Returns:
            MergeSuggestion with:
                - canonical_name: Suggested normalized name
                - rules_to_create: List of NormalizationRule objects
                - transactions_affected: Count of transactions that would change

        Example:
            >>> suggestion = engine.suggest_merge("cluster_amazon_1", "user_darwin")
            >>> suggestion.canonical_name
            "Amazon"
            >>> len(suggestion.rules_to_create)
            3  # One rule per variant
            >>> suggestion.transactions_affected
            68
        """

    def split_cluster(self, cluster_id: str, variants_to_extract: List[str],
                      new_canonical: str) -> Tuple[Cluster, Cluster]:
        """
        Splits cluster into two by extracting variants.

        Args:
            cluster_id: Original cluster ID
            variants_to_extract: Variant names to move to new cluster
            new_canonical: Canonical name for new cluster

        Returns:
            Tuple of (original_cluster, new_cluster)

        Raises:
            ClusterNotFoundError: If cluster_id invalid
            InvalidVariantError: If variant not in cluster

        Example:
            >>> # Split "Amazon" cluster, extract "Amazon Web Services"
            >>> original, new = engine.split_cluster(
            ...     "cluster_amazon_1",
            ...     ["Amazon Web Services"],
            ...     "Amazon Web Services"
            ... )
            >>> len(original.variants)
            3  # AMAZON.COM, AMZN MKTP US, AMAZON PRIME
            >>> len(new.variants)
            1  # Amazon Web Services
        """

    def merge_clusters(self, cluster_ids: List[str], new_canonical: str) -> Cluster:
        """
        Merges multiple clusters into one.

        Args:
            cluster_ids: List of cluster IDs to merge
            new_canonical: Canonical name for merged cluster

        Returns:
            New Cluster object with all variants combined

        Example:
            >>> # Merge "Amazon" and "Amazon Prime" clusters
            >>> merged = engine.merge_clusters(
            ...     ["cluster_amazon_1", "cluster_prime_1"],
            ...     "Amazon"
            ... )
            >>> len(merged.variants)
            5  # All variants combined
        """

    def find_canonical_name(self, variants: List[ClusterVariant]) -> str:
        """
        Determines canonical (representative) name for cluster.

        Strategy:
        1. Prefer most frequent variant (highest transaction count)
        2. If tie, prefer shortest clean name (no special chars)
        3. If still tie, prefer alphabetically first

        Args:
            variants: List of ClusterVariant objects

        Returns:
            Canonical name string

        Example:
            >>> variants = [
            ...     ClusterVariant("AMAZON.COM", frequency=23),
            ...     ClusterVariant("AMZN MKTP US", frequency=45),
            ...     ClusterVariant("Amazon", frequency=12)
            ... ]
            >>> engine.find_canonical_name(variants)
            "AMZN MKTP US"  # Highest frequency (45)
        """

    def detect_outliers(self, cluster: Cluster, threshold: float = 0.60) -> List[str]:
        """
        Detects outlier variants in cluster (low similarity to canonical).

        Args:
            cluster: Cluster to analyze
            threshold: Min similarity to canonical (default 0.60)

        Returns:
            List of variant names below threshold

        Example:
            >>> cluster = Cluster(
            ...     canonical_name="Amazon",
            ...     variants=[
            ...         ClusterVariant("AMAZON.COM", similarity=0.92),
            ...         ClusterVariant("Amazon Web Services", similarity=0.55)
            ...     ]
            ... )
            >>> engine.detect_outliers(cluster, threshold=0.60)
            ["Amazon Web Services"]  # Similarity 0.55 < 0.60
        """

    def visualize_cluster(self, cluster: Cluster) -> ClusterVisualization:
        """
        Generates visualization data for cluster (similarity heatmap).

        Args:
            cluster: Cluster to visualize

        Returns:
            ClusterVisualization with:
                - similarity_matrix: Pairwise similarity between variants
                - dendrogram: Hierarchical clustering tree (if algorithm=hierarchical)
                - outliers: List of outlier variants

        Example:
            >>> viz = engine.visualize_cluster(cluster_amazon)
            >>> viz.similarity_matrix[0][1]  # AMAZON.COM vs AMZN MKTP US
            0.78
        """
```

---

## Data Model

### Cluster

```python
@dataclass
class Cluster:
    """
    Cluster of similar items.

    Attributes:
        cluster_id: Unique identifier (e.g., "cluster_amazon_1")
        user_id: User who owns cluster
        canonical_name: Representative name (e.g., "Amazon")
        variants: List of ClusterVariant objects
        similarity_score: Average similarity within cluster [0.0, 1.0]
        transaction_count: Total transactions across all variants
        status: "pending" | "accepted" | "rejected" | "split"
        created_at: Creation timestamp
        updated_at: Last update timestamp
    """
    cluster_id: str
    user_id: str
    canonical_name: str
    variants: List[ClusterVariant]
    similarity_score: float
    transaction_count: int
    status: str
    created_at: datetime
    updated_at: datetime
```

### ClusterVariant

```python
@dataclass
class ClusterVariant:
    """
    Variant within cluster.

    Attributes:
        raw_name: Original text (e.g., "AMAZON.COM")
        frequency: Count of occurrences
        similarity_to_canonical: Similarity to canonical_name [0.0, 1.0]
    """
    raw_name: str
    frequency: int
    similarity_to_canonical: float
```

### MergeSuggestion

```python
@dataclass
class MergeSuggestion:
    """
    Suggestion to merge cluster and create normalization rules.

    Attributes:
        cluster_id: Cluster to merge
        canonical_name: Suggested normalized name
        rules_to_create: List of NormalizationRule objects (one per variant)
        transactions_affected: Count of transactions that would change
        confidence: Confidence score [0.0, 1.0] based on similarity
    """
    cluster_id: str
    canonical_name: str
    rules_to_create: List[NormalizationRule]
    transactions_affected: int
    confidence: float
```

### ClusterVisualization

```python
@dataclass
class ClusterVisualization:
    """
    Visualization data for cluster.

    Attributes:
        similarity_matrix: NxN matrix of pairwise similarities
        dendrogram: Hierarchical clustering tree (optional)
        outliers: List of variant names with low similarity
    """
    similarity_matrix: np.ndarray
    dendrogram: Optional[Dict]
    outliers: List[str]
```

---

## Algorithms

### DBSCAN (Density-Based Spatial Clustering)

**Characteristics:**
- Handles noise/outliers well
- No need to specify number of clusters
- Works with arbitrary cluster shapes

**Process:**
```
1. For each item:
   a. Find all neighbors within similarity threshold (eps)
   b. If neighbors >= min_samples, start cluster
   c. Recursively add neighbors' neighbors
2. Items not in any cluster = outliers
3. OUTPUT: List of clusters + outliers
```

**Example:**
```python
# Items: ["AMAZON.COM", "AMZN MKTP US", "AMAZON PRIME", "WALMART", "WAL-MART"]
# Threshold: 0.70
# Result:
# - Cluster 1: ["AMAZON.COM", "AMZN MKTP US", "AMAZON PRIME"]
# - Cluster 2: ["WALMART", "WAL-MART"]
```

### Hierarchical Clustering (Agglomerative)

**Characteristics:**
- Builds tree structure (dendrogram)
- Can cut tree at any level for different granularities
- Good for visualizing relationships

**Process:**
```
1. Start with each item as own cluster
2. Repeat until single cluster:
   a. Find two closest clusters
   b. Merge them
   c. Update distance matrix
3. Cut tree at desired similarity threshold
4. OUTPUT: List of clusters
```

**Example:**
```python
# Dendrogram:
#          Amazon (root)
#          /     \
#     0.92/       \0.78
#        /         \
#  AMAZON.COM   AMZN MKTP
#                    |
#                0.85|
#                    |
#              AMAZON PRIME
```

### Simple Threshold Clustering

**Characteristics:**
- Fast and simple
- Less accurate (greedy approach)
- Good for large datasets

**Process:**
```
1. Sort items alphabetically
2. For each item:
   a. Compare to all existing clusters
   b. If similarity > threshold, add to cluster
   c. Else, create new cluster
3. OUTPUT: List of clusters
```

---

## Multi-Domain Examples

### Finance: Merchant Name Clustering

```python
merchants = [
    "AMAZON.COM",
    "AMZN MKTP US",
    "AMAZON PRIME",
    "Amazon Web Services",
    "WALMART",
    "WAL-MART SUPERCENTER",
    "UBER EATS",
    "UBER *RIDE"
]

clusters = engine.create_clusters(
    merchants,
    similarity_threshold=0.70,
    algorithm="dbscan"
)

# Result:
# Cluster 1: Amazon (4 variants, 88 transactions)
#   - AMAZON.COM (similarity: 0.92)
#   - AMZN MKTP US (similarity: 0.78)
#   - AMAZON PRIME (similarity: 0.85)
#   - Amazon Web Services (similarity: 0.65) ← outlier
#
# Cluster 2: Walmart (2 variants, 34 transactions)
#   - WALMART (similarity: 0.88)
#   - WAL-MART SUPERCENTER (similarity: 0.88)
#
# Cluster 3: Uber (2 variants, 23 transactions)
#   - UBER EATS (similarity: 0.91)
#   - UBER *RIDE (similarity: 0.91)

# Detect outlier
outliers = engine.detect_outliers(clusters[0], threshold=0.70)
# → ["Amazon Web Services"]  # Similarity 0.65 < 0.70

# Split cluster
original, new = engine.split_cluster(
    clusters[0].cluster_id,
    ["Amazon Web Services"],
    "Amazon Web Services"
)
```

### Healthcare: Provider Name Clustering

```python
providers = [
    "ST MARY'S HOSP",
    "ST MARY'S HOSPITAL",
    "ST MARYS MEDICAL CENTER",
    "KAISER PERMANENTE",
    "KAISER PERM HOSP"
]

clusters = engine.create_clusters(
    providers,
    similarity_threshold=0.75,
    algorithm="hierarchical"
)

# Result:
# Cluster 1: St. Mary's (3 variants)
#   - ST MARY'S HOSP (similarity: 0.95)
#   - ST MARY'S HOSPITAL (similarity: 0.95)
#   - ST MARYS MEDICAL CENTER (similarity: 0.82)
#
# Cluster 2: Kaiser (2 variants)
#   - KAISER PERMANENTE (similarity: 0.88)
#   - KAISER PERM HOSP (similarity: 0.88)

# Visualize
viz = engine.visualize_cluster(clusters[0])
# Shows dendrogram with merge points
```

### Legal: Case Name Clustering

```python
cases = [
    "DOE V. SMITH",
    "Doe v. Smith",
    "JOHNSON V. ACME CORP",
    "Johnson v. ACME Corporation"
]

clusters = engine.create_clusters(
    cases,
    similarity_threshold=0.80,
    algorithm="simple"
)

# Result:
# Cluster 1: Doe v. Smith (2 variants)
# Cluster 2: Johnson v. ACME (2 variants)
```

### Research: Author Name Clustering

```python
authors = [
    "J. Smith",
    "John Smith",
    "Smith, J.",
    "Smith, John",
    "J. R. Smith"
]

clusters = engine.create_clusters(
    authors,
    similarity_threshold=0.70,
    algorithm="dbscan"
)

# Result:
# Cluster 1: John Smith (5 variants)
#   - All variants grouped despite different formats
```

---

## Performance Characteristics

### Latency Targets

- `create_clusters()` 100 items: <1s p95
- `create_clusters()` 1000 items: <3s p95
- `calculate_similarity_matrix()` 100x100: <500ms p95

### Optimization Strategies

1. **Similarity Matrix Caching:**
   - Cache matrix for reuse across algorithms
   - Invalidate when items change

2. **Parallel Similarity Calculation:**
   - Use multiprocessing for large datasets
   - Split matrix calculation across CPU cores

3. **Approximate Algorithms:**
   - Use MinHash for very large datasets (>10k items)
   - Trade accuracy for speed

### Scalability

- Items per cluster: Up to 50,000
- Clusters per user: Unlimited
- Background job: Process all merchants nightly

---

## Error Handling

### Cluster Not Found

```python
class ClusterNotFoundError(Exception):
    """Raised when cluster_id invalid."""
    pass

# Example
cluster = cluster_store.get(cluster_id)
if not cluster:
    raise ClusterNotFoundError(f"Cluster {cluster_id} not found")
```

### Invalid Variant

```python
class InvalidVariantError(Exception):
    """Raised when variant not in cluster."""
    pass

# Example
if variant not in cluster.variants:
    raise InvalidVariantError(f"Variant '{variant}' not in cluster")
```

---

## Testing

### Unit Tests

```python
def test_create_clusters_dbscan():
    items = ["AMAZON.COM", "AMZN MKTP US", "WALMART"]
    clusters = engine.create_clusters(items, similarity_threshold=0.70, algorithm="dbscan")
    assert len(clusters) == 2  # Amazon cluster + Walmart cluster
    amazon = [c for c in clusters if "AMAZON" in c.canonical_name][0]
    assert len(amazon.variants) == 2

def test_split_cluster():
    cluster = create_cluster(["AMAZON.COM", "Amazon Web Services"])
    original, new = engine.split_cluster(
        cluster.cluster_id,
        ["Amazon Web Services"],
        "AWS"
    )
    assert len(original.variants) == 1
    assert len(new.variants) == 1
    assert new.canonical_name == "AWS"

def test_detect_outliers():
    cluster = Cluster(
        canonical_name="Amazon",
        variants=[
            ClusterVariant("AMAZON.COM", frequency=50, similarity_to_canonical=0.92),
            ClusterVariant("Amazon Locker", frequency=5, similarity_to_canonical=0.55)
        ]
    )
    outliers = engine.detect_outliers(cluster, threshold=0.70)
    assert "Amazon Locker" in outliers

def test_find_canonical_name_highest_frequency():
    variants = [
        ClusterVariant("AMAZON.COM", frequency=23, similarity_to_canonical=0.92),
        ClusterVariant("AMZN MKTP US", frequency=45, similarity_to_canonical=0.78)
    ]
    canonical = engine.find_canonical_name(variants)
    assert canonical == "AMZN MKTP US"  # Highest frequency
```

---

## Security Considerations

### Resource Limits

- Max items per cluster: 50,000 (prevent DoS)
- Timeout clustering job at 10 minutes
- Max similarity matrix size: 10,000 x 10,000

### Input Validation

- Validate similarity_threshold in [0.0, 1.0]
- Validate min_cluster_size >= 2
- Sanitize cluster names for XSS

---

## Observability

### Metrics

```
clustering.job.runtime (histogram, labels: item_count, algorithm)
clustering.clusters_created.count (counter, labels: user_id)
clustering.outliers_detected.count (counter, labels: cluster_id)
clustering.split.count (counter, labels: cluster_id)
```

### Structured Logs

```json
{
  "event": "clustering_job_complete",
  "user_id": "user_darwin",
  "items_processed": 1234,
  "clusters_created": 45,
  "outliers_detected": 12,
  "algorithm": "dbscan",
  "similarity_threshold": 0.70,
  "runtime_ms": 2500,
  "timestamp": "2025-10-24T02:00:00Z"
}
```

---

## Related Primitives

- **FuzzyMatcher** - Calculates similarity scores
- **MerchantNormalizer** - Applies normalization rules
- **NormalizationRuleStore** - Stores rules created from clusters

---

## Summary

**ClusteringEngine** provides similarity-based clustering with:
- ✅ Multiple algorithms (DBSCAN, hierarchical, simple)
- ✅ Auto-detection of entity variants
- ✅ Outlier detection
- ✅ Cluster split/merge operations
- ✅ Visualization data generation
- ✅ Multi-domain applicability (finance, healthcare, legal, research)

**Universal Pattern:**
Applicable to any domain requiring entity deduplication and variant grouping.
