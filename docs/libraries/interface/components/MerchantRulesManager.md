# Primitive: MerchantRulesManager (IL)

> **Layer:** Interaction Layer (IL)
> **Type:** CRUD Interface
> **Vertical:** 3.8 Cluster Rules
> **Last Updated:** 2025-10-24

---

## Overview

**MerchantRulesManager** is an Interaction Layer component that provides a full CRUD interface for managing normalization rules. Users can create, edit, delete, test, and preview rules before applying them.

**Core Responsibility:**
Provide intuitive UI for managing text normalization rules with priority ordering, rule testing, and batch operations.

**Domain Instantiation:**
- Finance: Manage merchant normalization rules
- Healthcare: Manage provider/medication normalization rules
- Legal: Manage case/party name normalization rules
- Research (RSRCH - Utilitario): Manage entity/source normalization rules (founder names, company names)

---

## Interface

### Props

```typescript
interface MerchantRulesManagerProps {
  userId: string;
  rules: NormalizationRule[];
  onCreateRule: (rule: CreateRuleRequest) => Promise<NormalizationRule>;
  onEditRule: (ruleId: string, updates: Partial<NormalizationRule>) => Promise<NormalizationRule>;
  onDeleteRule: (ruleId: string) => Promise<void>;
  onTestRule: (pattern: string, ruleType: RuleType, threshold?: number) => Promise<RuleTestResult>;
  onToggleRule: (ruleId: string, enabled: boolean) => Promise<void>;
  onBatchApply: (ruleIds: string[]) => Promise<BatchApplyResult>;
  isLoading?: boolean;
  error?: string;
}

interface NormalizationRule {
  rule_id: string;
  pattern: string;
  replacement: string;
  rule_type: 'exact' | 'regex' | 'fuzzy' | 'soundex';
  priority: number;
  similarity_threshold?: number;
  enabled: boolean;
  created_at: string;
  updated_at: string;
  match_count: number;
}

interface RuleTestResult {
  matches: Array<{
    raw_name: string;
    normalized_name: string;
    similarity: number;
  }>;
  total_transactions: number;
  sample_matches: Array<{
    raw_name: string;
    normalized_name: string;
    similarity: number;
  }>;
}

interface BatchApplyResult {
  normalized_count: number;
  rules_applied: Array<{
    rule_id: string;
    transaction_count: number;
  }>;
}
```

### States

```typescript
type ViewState =
  | 'list'        // Viewing rule list
  | 'creating'    // Creating new rule
  | 'editing'     // Editing existing rule
  | 'testing'     // Testing rule preview
  | 'deleting'    // Confirming delete
  | 'applying';   // Batch applying rules

interface ComponentState {
  viewState: ViewState;
  selectedRule: NormalizationRule | null;
  testResult: RuleTestResult | null;
  sortBy: 'priority' | 'created_at' | 'match_count';
  filterType: 'all' | 'exact' | 'regex' | 'fuzzy' | 'soundex';
  searchQuery: string;
}
```

---

## Wireframes

### Wireframe 1: Rule List View

```
┌─────────────────────────────────────────────────────────────────┐
│ Normalization Rules                      [+ Create Rule] [Test] │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ 🔍 Search rules...                 Sort: [Priority ▼] Type: [All▼]│
│                                                                   │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ Priority 95 │ EXACT │ ✅ Enabled         [Edit] [Delete] [⋮]│ │
│ │ Pattern: "AMZN MKTP US"                                     │ │
│ │ Replacement: "Amazon"                                       │ │
│ │ Matches: 45 transactions                                    │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ Priority 90 │ REGEX │ ✅ Enabled         [Edit] [Delete] [⋮]│ │
│ │ Pattern: "UBER.*"                                           │ │
│ │ Replacement: "Uber"                                         │ │
│ │ Matches: 47 transactions                                    │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ Priority 85 │ FUZZY │ ✅ Enabled         [Edit] [Delete] [⋮]│ │
│ │ Pattern: "McDonald's" (threshold: 0.80)                     │ │
│ │ Replacement: "McDonald's"                                   │ │
│ │ Matches: 23 transactions                                    │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│ Total: 3 rules │ Enabled: 3 │ Total matches: 115 transactions   │
└─────────────────────────────────────────────────────────────────┘
```

### Wireframe 2: Create/Edit Rule Dialog

```
┌───────────────────────────────────────────────────────────┐
│ Create Normalization Rule                  [×]            │
├───────────────────────────────────────────────────────────┤
│                                                             │
│ Rule Type: ● Exact  ○ Regex  ○ Fuzzy  ○ Soundex           │
│                                                             │
│ Pattern *                                                   │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ UBER.*                                               │   │
│ └─────────────────────────────────────────────────────┘   │
│ Enter the pattern to match (exact text, regex, or fuzzy)  │
│                                                             │
│ Replacement *                                               │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ Uber                                                 │   │
│ └─────────────────────────────────────────────────────┘   │
│ Normalized output (e.g., "Uber Eats")                     │
│                                                             │
│ Priority (0-100) *                                          │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ ──────────●──────────────── 90                       │   │
│ └─────────────────────────────────────────────────────┘   │
│ Higher priority rules execute first                        │
│                                                             │
│ [ ] Fuzzy Match Threshold (0.70-1.0)                      │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ 0.80                                                 │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│             [Test Rule]  [Cancel]  [Save Rule]             │
└───────────────────────────────────────────────────────────┘
```

### Wireframe 3: Rule Test Preview

```
┌───────────────────────────────────────────────────────────┐
│ Test Rule: "UBER.*" → "Uber"                  [×]          │
├───────────────────────────────────────────────────────────┤
│                                                             │
│ 💡 This rule will match 47 transactions                    │
│                                                             │
│ Sample Matches (showing 10 of 47):                         │
│                                                             │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ "UBER EATS PENDING" → "Uber"            ✅ Match    │   │
│ │ Similarity: 1.0 (regex)                             │   │
│ │ Transactions: 15                                    │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ "UBER *RIDE" → "Uber"                   ✅ Match    │   │
│ │ Similarity: 1.0 (regex)                             │   │
│ │ Transactions: 23                                    │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ "Uber Technologies" → "Uber"            ✅ Match    │   │
│ │ Similarity: 1.0 (regex)                             │   │
│ │ Transactions: 9                                     │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ ⚠️ Warning: This will overwrite any manual merchant edits │
│                                                             │
│              [Adjust Pattern]  [Cancel]  [Save & Apply]    │
└───────────────────────────────────────────────────────────┘
```

### Wireframe 4: Delete Confirmation

```
┌───────────────────────────────────────────────────────────┐
│ Delete Rule?                                   [×]          │
├───────────────────────────────────────────────────────────┤
│                                                             │
│ ⚠️ Are you sure you want to delete this rule?             │
│                                                             │
│ Pattern: "UBER.*"                                          │
│ Replacement: "Uber"                                        │
│ Matches: 47 transactions                                   │
│                                                             │
│ ⚠️ This rule has been applied to 47 transactions.         │
│ Deleting it will NOT revert past normalizations.          │
│                                                             │
│ Options:                                                    │
│ ● Disable rule (recommended - preserves history)          │
│ ○ Delete rule (permanent, cannot undo)                    │
│                                                             │
│                         [Cancel]  [Confirm Delete]         │
└───────────────────────────────────────────────────────────┘
```

---

## Component Behavior

### Create Rule Flow

```typescript
async function handleCreateRule() {
  // 1. Validate inputs
  if (!pattern || !replacement) {
    setError('Pattern and replacement are required');
    return;
  }

  if (ruleType === 'regex') {
    try {
      new RegExp(pattern);
    } catch (e) {
      setError('Invalid regex pattern');
      return;
    }
  }

  if (ruleType === 'fuzzy' && !similarityThreshold) {
    setError('Fuzzy rules require similarity threshold');
    return;
  }

  // 2. Test rule first (optional but recommended)
  const testResult = await onTestRule(pattern, ruleType, similarityThreshold);
  setTestResult(testResult);
  setViewState('testing');

  // User reviews test results...

  // 3. Create rule
  const rule = await onCreateRule({
    pattern,
    replacement,
    rule_type: ruleType,
    priority,
    similarity_threshold: similarityThreshold
  });

  // 4. Refresh rule list
  refetch();
  setViewState('list');
}
```

### Edit Rule Flow

```typescript
async function handleEditRule(ruleId: string) {
  // 1. Load rule
  const rule = rules.find(r => r.rule_id === ruleId);
  setSelectedRule(rule);
  setViewState('editing');

  // User makes changes...

  // 2. Save updates
  const updated = await onEditRule(ruleId, {
    pattern: newPattern,
    replacement: newReplacement,
    priority: newPriority
  });

  // 3. Refresh
  refetch();
  setViewState('list');
}
```

### Test Rule Flow

```typescript
async function handleTestRule() {
  setIsLoading(true);

  try {
    // Call API to test rule
    const result = await onTestRule(pattern, ruleType, similarityThreshold);

    // Show preview
    setTestResult(result);
    setViewState('testing');

    // User can see:
    // - Total matches
    // - Sample matches with similarity scores
    // - Transactions affected
  } catch (error) {
    setError(error.message);
  } finally {
    setIsLoading(false);
  }
}
```

### Delete Rule Flow

```typescript
async function handleDeleteRule(ruleId: string) {
  const rule = rules.find(r => r.rule_id === ruleId);

  // 1. Show confirmation
  setSelectedRule(rule);
  setViewState('deleting');

  // User chooses: disable or delete

  // 2. If disable (soft delete)
  if (action === 'disable') {
    await onToggleRule(ruleId, false);
  }

  // 3. If delete (hard delete)
  if (action === 'delete') {
    if (rule.match_count > 0) {
      setError('Cannot delete rule with matches. Disable instead.');
      return;
    }
    await onDeleteRule(ruleId);
  }

  // 4. Refresh
  refetch();
  setViewState('list');
}
```

---

## Multi-Domain Examples

### Finance: Merchant Rules Management

```tsx
<MerchantRulesManager
  userId="user_darwin"
  rules={merchantRules}
  onCreateRule={async (rule) => {
    // Create merchant normalization rule
    return api.post('/api/normalization-rules', rule);
  }}
  onTestRule={async (pattern, type, threshold) => {
    // Preview merchant matches
    return api.post('/api/normalization-rules/test', {
      pattern,
      rule_type: type,
      similarity_threshold: threshold
    });
  }}
/>
```

### Healthcare: Provider Rules Management

```tsx
<MerchantRulesManager
  userId="hospital_admin"
  rules={providerRules}
  onCreateRule={async (rule) => {
    // Create provider normalization rule
    return api.post('/api/provider-rules', rule);
  }}
/>
```

### Legal: Case Name Rules Management

```tsx
<MerchantRulesManager
  userId="law_firm"
  rules={caseNameRules}
  onCreateRule={async (rule) => {
    // Create case name normalization rule
    return api.post('/api/case-rules', rule);
  }}
/>
```

---

## Accessibility

### Keyboard Navigation

- `Tab` - Navigate between inputs
- `Enter` - Submit form / Save rule
- `Esc` - Close dialog / Cancel
- `Arrow Up/Down` - Navigate rule list
- `Space` - Toggle rule enabled/disabled

### Screen Reader Support

```tsx
<div role="region" aria-label="Normalization Rules Manager">
  <h2 id="rules-heading">Normalization Rules</h2>
  <ul aria-labelledby="rules-heading">
    {rules.map(rule => (
      <li key={rule.rule_id} aria-label={`Rule: ${rule.pattern} to ${rule.replacement}`}>
        <span aria-label={`Priority ${rule.priority}`}>{rule.priority}</span>
        <span aria-label={`Type: ${rule.rule_type}`}>{rule.rule_type}</span>
        <button aria-label={`Edit rule ${rule.pattern}`}>Edit</button>
        <button aria-label={`Delete rule ${rule.pattern}`}>Delete</button>
      </li>
    ))}
  </ul>
</div>
```

### Color Contrast

- Enabled rules: Green badge (#10B981) - WCAG AA compliant
- Disabled rules: Gray badge (#6B7280) - WCAG AA compliant
- Priority slider: High contrast track and thumb

---

## Performance

### Optimizations

1. **Virtualized List:**
   - Render only visible rules (react-window)
   - Handle 1000+ rules smoothly

2. **Debounced Search:**
   - Debounce search input (300ms)
   - Avoid excessive filtering

3. **Optimistic Updates:**
   - Update UI immediately on toggle
   - Rollback on error

4. **Cached Test Results:**
   - Cache test results for 5 minutes
   - Re-test only if pattern changes

---

## Related Components

- **RuleEditorDialog** - Create/edit rule form
- **ClusterViewer** - View cluster suggestions
- **TransactionList** - View normalized transactions

---

## Summary

**MerchantRulesManager** provides:
- ✅ Full CRUD interface for normalization rules
- ✅ Rule testing with preview
- ✅ Priority management with slider
- ✅ Soft/hard delete with confirmation
- ✅ Search and filter capabilities
- ✅ Multi-domain support (finance, healthcare, legal, research)

**Universal Pattern:**
Applicable to any domain requiring user-defined text transformation rules.
