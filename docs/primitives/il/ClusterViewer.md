# Primitive: ClusterViewer (IL)

> **Layer:** Interaction Layer (IL)
> **Type:** Data Visualization
> **Vertical:** 3.8 Cluster Rules
> **Last Updated:** 2025-10-24

---

## Overview

**ClusterViewer** is an Interaction Layer component that displays clusters of similar entities (merchants, providers, institutions) with similarity scores, allowing users to accept merges, reject suggestions, or split clusters.

**Core Responsibility:**
Visualize entity clusters, show similarity heatmaps, and enable user-driven merge/split operations for entity deduplication.

**Domain Instantiation:**
- Finance: View merchant name clusters for normalization
- Healthcare: View provider name clusters for deduplication
- Legal: View case name clusters for matter grouping
- Research: View author/institution clusters for citation management

---

## Interface

### Props

```typescript
interface ClusterViewerProps {
  userId: string;
  clusters: MerchantCluster[];
  onAcceptMerge: (clusterId: string) => Promise<AcceptMergeResult>;
  onRejectCluster: (clusterId: string) => Promise<void>;
  onSplitCluster: (clusterId: string, variantsToExtract: string[], newCanonical: string) => Promise<SplitResult>;
  onViewCluster: (clusterId: string) => void;
  minSimilarity?: number;
  minTransactions?: number;
  isLoading?: boolean;
  error?: string;
}

interface MerchantCluster {
  cluster_id: string;
  canonical_name: string;
  variants: ClusterVariant[];
  similarity_score: number;
  transaction_count: number;
  status: 'pending' | 'accepted' | 'rejected' | 'split';
  created_at: string;
}

interface ClusterVariant {
  raw_name: string;
  frequency: number;
  similarity_to_canonical: number;
}

interface AcceptMergeResult {
  rules_created: Array<{ rule_id: string; pattern: string; replacement: string }>;
  transactions_normalized: number;
}

interface SplitResult {
  original_cluster_id: string;
  new_cluster_id: string;
  transactions_moved: number;
}
```

### States

```typescript
type ViewState =
  | 'overview'     // Grid of all clusters
  | 'detail'       // Single cluster detail
  | 'merging'      // Accepting merge
  | 'splitting'    // Splitting cluster
  | 'rejecting';   // Rejecting cluster

interface ComponentState {
  viewState: ViewState;
  selectedCluster: MerchantCluster | null;
  selectedVariants: string[];
  sortBy: 'similarity' | 'transaction_count' | 'created_at';
  filterStatus: 'all' | 'pending' | 'accepted' | 'rejected';
}
```

---

## Wireframes

### Wireframe 1: Cluster Overview

```
┌─────────────────────────────────────────────────────────────────┐
│ Merchant Clusters                        Min Similarity: [0.70▼] │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Sort: [Similarity ▼]  Status: [Pending ▼]  Min Txns: [5▼]       │
│                                                                   │
│ ┌────────────────────────┐  ┌────────────────────────┐          │
│ │ Amazon                  │  │ Walmart                │          │
│ │ 📊 Similarity: 0.85     │  │ 📊 Similarity: 0.88    │          │
│ │ 💰 88 transactions      │  │ 💰 34 transactions     │          │
│ │                         │  │                        │          │
│ │ 4 variants:             │  │ 2 variants:            │          │
│ │ • AMAZON.COM (23)       │  │ • WALMART (20)         │          │
│ │ • AMZN MKTP US (45)     │  │ • WAL-MART (14)        │          │
│ │ • AMAZON PRIME (12)     │  │                        │          │
│ │ • Amazon Web Svcs (8)   │  │ ⚠️ 1 outlier          │          │
│ │                         │  │                        │          │
│ │ ⚠️ 1 outlier detected  │  │ [View] [Accept] [×]    │          │
│ │                         │  │                        │          │
│ │ [View] [Accept] [×]     │  └────────────────────────┘          │
│ └────────────────────────┘                                        │
│                                                                   │
│ ┌────────────────────────┐  ┌────────────────────────┐          │
│ │ Uber                    │  │ McDonald's             │          │
│ │ 📊 Similarity: 0.91     │  │ 📊 Similarity: 0.87    │          │
│ │ 💰 23 transactions      │  │ 💰 23 transactions     │          │
│ │                         │  │                        │          │
│ │ 2 variants:             │  │ 3 variants:            │          │
│ │ • UBER EATS (15)        │  │ • MCDONALDS (15)       │          │
│ │ • UBER *RIDE (8)        │  │ • MCDONALD'S (8)       │          │
│ │                         │  │                        │          │
│ │ [View] [Accept] [×]     │  │ [View] [Accept] [×]    │          │
│ └────────────────────────┘  └────────────────────────┘          │
│                                                                   │
│ Total: 4 clusters │ Pending: 4 │ Total transactions: 168         │
└─────────────────────────────────────────────────────────────────┘
```

### Wireframe 2: Cluster Detail View

```
┌─────────────────────────────────────────────────────────────────┐
│ Cluster: Amazon                            [×] Close             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Canonical Name: Amazon                                           │
│ Overall Similarity: 0.85 (Good)                                  │
│ Total Transactions: 88                                           │
│ Status: Pending Review                                           │
│                                                                   │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ Variants & Similarity Heatmap                                │ │
│ ├─────────────────────────────────────────────────────────────┤ │
│ │                                                              │ │
│ │ ✅ AMAZON.COM (23 txns)                                      │ │
│ │    Similarity to "Amazon": 0.92 ████████████████████░░       │ │
│ │    [ ] Split into new cluster                                │ │
│ │                                                              │ │
│ │ ✅ AMZN MKTP US (45 txns)                                    │ │
│ │    Similarity to "Amazon": 0.78 ███████████████░░░░░         │ │
│ │    [ ] Split into new cluster                                │ │
│ │                                                              │ │
│ │ ✅ AMAZON PRIME (12 txns)                                    │ │
│ │    Similarity to "Amazon": 0.85 ████████████████░░░░         │ │
│ │    [ ] Split into new cluster                                │ │
│ │                                                              │ │
│ │ ⚠️ Amazon Web Services (8 txns) - OUTLIER                   │ │
│ │    Similarity to "Amazon": 0.65 ████████████░░░░░░░░░        │ │
│ │    [✓] Split into new cluster                                │ │
│ │                                                              │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│ 💡 Recommendation: Split "Amazon Web Services" (low similarity) │
│                                                                   │
│ [Split Selected] [Reject Cluster] [Accept Merge]                │
└─────────────────────────────────────────────────────────────────┘
```

### Wireframe 3: Accept Merge Confirmation

```
┌───────────────────────────────────────────────────────────┐
│ Accept Cluster Merge?                          [×]         │
├───────────────────────────────────────────────────────────┤
│                                                             │
│ This will create normalization rules for:                  │
│                                                             │
│ ✅ "AMAZON.COM" → "Amazon"                                 │
│ ✅ "AMZN MKTP US" → "Amazon"                               │
│ ✅ "AMAZON PRIME" → "Amazon"                               │
│                                                             │
│ Impact:                                                     │
│ • 3 new normalization rules (priority 80)                  │
│ • 80 transactions will be normalized to "Amazon"          │
│ • Rules will auto-apply to future transactions            │
│                                                             │
│ ⚠️ Note: "Amazon Web Services" will remain separate       │
│                                                             │
│                    [Cancel]  [Confirm Merge]               │
└───────────────────────────────────────────────────────────┘
```

### Wireframe 4: Split Cluster Dialog

```
┌───────────────────────────────────────────────────────────┐
│ Split Cluster                                  [×]         │
├───────────────────────────────────────────────────────────┤
│                                                             │
│ Variants to Extract:                                        │
│ ✅ Amazon Web Services (8 transactions)                    │
│                                                             │
│ New Canonical Name *                                        │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ Amazon Web Services                                  │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ Preview:                                                    │
│ Original Cluster: "Amazon" (80 transactions, 3 variants)   │
│ New Cluster: "Amazon Web Services" (8 transactions, 1 var) │
│                                                             │
│                         [Cancel]  [Confirm Split]          │
└───────────────────────────────────────────────────────────┘
```

---

## Component Behavior

### Accept Merge Flow

```typescript
async function handleAcceptMerge(clusterId: string) {
  setViewState('merging');

  try {
    // Call API to accept merge
    const result = await onAcceptMerge(clusterId);

    // Show success notification
    toast.success(`✅ Created ${result.rules_created.length} rules, normalized ${result.transactions_normalized} transactions`);

    // Update cluster status
    updateClusterStatus(clusterId, 'accepted');

    // Refresh clusters
    refetch();
  } catch (error) {
    toast.error(error.message);
  } finally {
    setViewState('overview');
  }
}
```

### Split Cluster Flow

```typescript
async function handleSplitCluster() {
  // 1. Validate selections
  if (selectedVariants.length === 0) {
    setError('Select at least one variant to split');
    return;
  }

  if (!newCanonicalName) {
    setError('Enter canonical name for new cluster');
    return;
  }

  // 2. Confirm split
  setViewState('splitting');

  try {
    const result = await onSplitCluster(
      selectedCluster.cluster_id,
      selectedVariants,
      newCanonicalName
    );

    // 3. Show success
    toast.success(`✅ Split cluster, moved ${result.transactions_moved} transactions`);

    // 4. Refresh
    refetch();
    setViewState('overview');
  } catch (error) {
    toast.error(error.message);
  }
}
```

### Reject Cluster Flow

```typescript
async function handleRejectCluster(clusterId: string) {
  // Confirm rejection
  const confirmed = confirm('Are you sure you want to reject this cluster suggestion?');
  if (!confirmed) return;

  setViewState('rejecting');

  try {
    await onRejectCluster(clusterId);

    // Update status
    updateClusterStatus(clusterId, 'rejected');

    // Remove from view (if filtering by pending)
    if (filterStatus === 'pending') {
      setClusters(clusters.filter(c => c.cluster_id !== clusterId));
    }

    toast.success('Cluster suggestion rejected');
  } catch (error) {
    toast.error(error.message);
  } finally {
    setViewState('overview');
  }
}
```

---

## Multi-Domain Examples

### Finance: Merchant Clusters

```tsx
<ClusterViewer
  userId="user_darwin"
  clusters={merchantClusters}
  onAcceptMerge={async (clusterId) => {
    return api.post(`/api/merchant-clusters/${clusterId}/accept`);
  }}
  onSplitCluster={async (clusterId, variants, canonical) => {
    return api.post(`/api/merchant-clusters/${clusterId}/split`, {
      variants_to_extract: variants,
      new_canonical_name: canonical
    });
  }}
  minSimilarity={0.70}
  minTransactions={5}
/>
```

### Healthcare: Provider Clusters

```tsx
<ClusterViewer
  userId="hospital_admin"
  clusters={providerClusters}
  onAcceptMerge={async (clusterId) => {
    return api.post(`/api/provider-clusters/${clusterId}/accept`);
  }}
  minSimilarity={0.75}
/>
```

### Research: Author Clusters

```tsx
<ClusterViewer
  userId="researcher"
  clusters={authorClusters}
  onAcceptMerge={async (clusterId) => {
    return api.post(`/api/author-clusters/${clusterId}/accept`);
  }}
  minSimilarity={0.70}
/>
```

---

## Accessibility

### Keyboard Navigation

- `Tab` - Navigate between clusters
- `Enter` - View cluster detail
- `Space` - Select/deselect variant
- `Esc` - Close detail view
- `Arrow keys` - Navigate variant list

### Screen Reader Support

```tsx
<div role="region" aria-label="Merchant Clusters">
  <h2 id="clusters-heading">Merchant Clusters</h2>
  <div aria-labelledby="clusters-heading" role="grid">
    {clusters.map(cluster => (
      <div
        key={cluster.cluster_id}
        role="gridcell"
        aria-label={`Cluster: ${cluster.canonical_name}, ${cluster.variants.length} variants, ${cluster.transaction_count} transactions, similarity ${cluster.similarity_score}`}
      >
        <button aria-label={`View ${cluster.canonical_name} cluster details`}>
          View
        </button>
        <button aria-label={`Accept merge for ${cluster.canonical_name}`}>
          Accept
        </button>
      </div>
    ))}
  </div>
</div>
```

---

## Performance

### Optimizations

1. **Virtualized Grid:**
   - Render only visible clusters
   - Handle 100+ clusters smoothly

2. **Lazy Load Details:**
   - Load cluster details on demand
   - Only when user clicks "View"

3. **Memoized Similarity:**
   - Cache similarity calculations
   - Avoid recomputing on re-render

---

## Related Components

- **MerchantRulesManager** - Manage created rules
- **RuleEditorDialog** - Edit auto-created rules
- **TransactionList** - View normalized transactions

---

## Summary

**ClusterViewer** provides:
- ✅ Visual cluster overview with similarity scores
- ✅ Similarity heatmap for variants
- ✅ Accept merge (auto-create rules)
- ✅ Split cluster (extract outliers)
- ✅ Reject suggestions
- ✅ Multi-domain support (finance, healthcare, legal, research)

**Universal Pattern:**
Applicable to any domain requiring entity deduplication and variant grouping.
