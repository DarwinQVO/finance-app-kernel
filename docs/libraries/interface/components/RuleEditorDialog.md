# Primitive: RuleEditorDialog (IL)

> **Layer:** Interaction Layer (IL)
> **Type:** Modal Dialog
> **Vertical:** 3.8 Cluster Rules
> **Last Updated:** 2025-10-24

---

## Overview

**RuleEditorDialog** is an Interaction Layer component that provides a modal dialog for creating and editing normalization rules. It includes pattern input, rule type selection, priority slider, fuzzy threshold configuration, and real-time rule testing.

**Core Responsibility:**
Provide intuitive form-based interface for creating/editing text normalization rules with validation, testing, and preview capabilities.

**Domain Instantiation:**
- Finance: Create merchant normalization rules
- Healthcare: Create provider/medication normalization rules
- Legal: Create case/party name normalization rules
- Research: Create institution/author normalization rules

---

## Interface

### Props

```typescript
interface RuleEditorDialogProps {
  isOpen: boolean;
  mode: 'create' | 'edit';
  rule?: NormalizationRule;  // Required if mode='edit'
  onSave: (rule: CreateRuleRequest | UpdateRuleRequest) => Promise<NormalizationRule>;
  onCancel: () => void;
  onTest: (pattern: string, ruleType: RuleType, threshold?: number) => Promise<RuleTestResult>;
  isLoading?: boolean;
  error?: string;
}

interface CreateRuleRequest {
  pattern: string;
  replacement: string;
  rule_type: 'exact' | 'regex' | 'fuzzy' | 'soundex';
  priority: number;
  similarity_threshold?: number;
}

interface UpdateRuleRequest extends Partial<CreateRuleRequest> {
  rule_id: string;
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
```

### States

```typescript
type DialogState =
  | 'editing'   // User editing form
  | 'testing'   // Previewing matches
  | 'saving'    // Saving rule
  | 'error';    // Validation error

interface ComponentState {
  dialogState: DialogState;
  pattern: string;
  replacement: string;
  ruleType: 'exact' | 'regex' | 'fuzzy' | 'soundex';
  priority: number;
  similarityThreshold?: number;
  testResult: RuleTestResult | null;
  validationErrors: Record<string, string>;
}
```

---

## Wireframe

```
┌───────────────────────────────────────────────────────────┐
│ Create Normalization Rule                  [×]            │
├───────────────────────────────────────────────────────────┤
│                                                             │
│ Rule Type: ● Exact  ○ Regex  ○ Fuzzy  ○ Soundex           │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ ℹ️ Exact: Match exact string (case-insensitive)     │   │
│ │ Regex: Match pattern (e.g., "UBER.*")               │   │
│ │ Fuzzy: Match similar strings (threshold-based)      │   │
│ │ Soundex: Match similar pronunciation                │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ Pattern *                                                   │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ UBER.*                                               │   │
│ └─────────────────────────────────────────────────────┘   │
│ ⚠️ Regex pattern - use .* for wildcard                    │
│                                                             │
│ Replacement *                                               │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ Uber                                                 │   │
│ └─────────────────────────────────────────────────────┘   │
│ This will be the normalized name in transactions          │
│                                                             │
│ Priority (0-100) *                                          │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ 0────────────●──────────────────────────────── 100   │   │
│ │                    90                                │   │
│ └─────────────────────────────────────────────────────┘   │
│ Higher priority rules execute first (exact > regex > fuzzy)│
│                                                             │
│ [ ] Fuzzy Match Settings (only for fuzzy type)            │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ Similarity Threshold (0.70-1.0)                      │   │
│ │ ┌───────────────────────────────────────────────┐   │   │
│ │ │ 0.70──────────●────────────────────────── 1.0 │   │   │
│ │ │                  0.80                         │   │   │
│ │ └───────────────────────────────────────────────┘   │   │
│ │ ℹ️ Lower threshold = more matches (less strict)     │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ 💡 Test Results (47 transactions matched)            │   │
│ │ ┌─────────────────────────────────────────────────┐ │   │
│ │ │ "UBER EATS PENDING" → "Uber" ✅ (15 txns)       │ │   │
│ │ │ "UBER *RIDE" → "Uber" ✅ (23 txns)              │ │   │
│ │ │ "Uber Technologies" → "Uber" ✅ (9 txns)        │ │   │
│ │ └─────────────────────────────────────────────────┘ │   │
│ │ [View All Matches]                                   │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│             [Test Rule]  [Cancel]  [Save Rule]             │
└───────────────────────────────────────────────────────────┘
```

---

## Component Behavior

### Validation Logic

```typescript
function validateForm(): Record<string, string> {
  const errors: Record<string, string> = {};

  // Pattern validation
  if (!pattern || pattern.trim().length === 0) {
    errors.pattern = 'Pattern is required';
  } else if (pattern.length > 500) {
    errors.pattern = 'Pattern must be ≤500 characters';
  } else if (ruleType === 'regex') {
    try {
      new RegExp(pattern);
    } catch (e) {
      errors.pattern = 'Invalid regex pattern';
    }

    // Check for catastrophic backtracking
    if (/(a\+)\+|(a\*)\*|(a\|a)\*/.test(pattern)) {
      errors.pattern = 'Pattern may cause catastrophic backtracking';
    }
  } else if (ruleType === 'fuzzy' && pattern.length < 3) {
    errors.pattern = 'Fuzzy pattern must be at least 3 characters';
  }

  // Replacement validation
  if (!replacement || replacement.trim().length === 0) {
    errors.replacement = 'Replacement is required';
  } else if (replacement.length > 200) {
    errors.replacement = 'Replacement must be ≤200 characters';
  }

  // Priority validation
  if (priority < 0 || priority > 100) {
    errors.priority = 'Priority must be between 0 and 100';
  }

  // Fuzzy threshold validation
  if (ruleType === 'fuzzy') {
    if (similarityThreshold === undefined) {
      errors.similarityThreshold = 'Fuzzy rules require similarity threshold';
    } else if (similarityThreshold < 0.0 || similarityThreshold > 1.0) {
      errors.similarityThreshold = 'Threshold must be between 0.0 and 1.0';
    } else if (similarityThreshold < 0.70) {
      errors.similarityThreshold = 'Threshold should be ≥0.70 (recommended)';
    }
  }

  return errors;
}
```

### Test Rule Flow

```typescript
async function handleTestRule() {
  // 1. Validate inputs
  const errors = validateForm();
  if (Object.keys(errors).length > 0) {
    setValidationErrors(errors);
    return;
  }

  // 2. Call test API
  setDialogState('testing');

  try {
    const result = await onTest(pattern, ruleType, similarityThreshold);

    // 3. Show results
    setTestResult(result);

    // 4. Provide feedback
    if (result.total_transactions === 0) {
      toast.warning('⚠️ No transactions match this pattern');
    } else {
      toast.success(`✅ ${result.total_transactions} transactions matched`);
    }
  } catch (error) {
    setDialogState('error');
    setError(error.message);
  }
}
```

### Save Rule Flow

```typescript
async function handleSave() {
  // 1. Validate
  const errors = validateForm();
  if (Object.keys(errors).length > 0) {
    setValidationErrors(errors);
    return;
  }

  // 2. Confirm if large impact
  if (testResult && testResult.total_transactions > 100) {
    const confirmed = confirm(
      `This will affect ${testResult.total_transactions} transactions. Continue?`
    );
    if (!confirmed) return;
  }

  // 3. Save
  setDialogState('saving');

  try {
    const request = mode === 'create'
      ? {
          pattern,
          replacement,
          rule_type: ruleType,
          priority,
          similarity_threshold: similarityThreshold
        }
      : {
          rule_id: rule.rule_id,
          pattern,
          replacement,
          priority,
          similarity_threshold: similarityThreshold
        };

    const saved = await onSave(request);

    // 4. Success
    toast.success(mode === 'create' ? '✅ Rule created' : '✅ Rule updated');
    onCancel();  // Close dialog
  } catch (error) {
    setDialogState('error');
    setError(error.message);
  }
}
```

### Rule Type Change Handler

```typescript
function handleRuleTypeChange(newType: RuleType) {
  setRuleType(newType);

  // Update priority based on type (best practice)
  if (newType === 'exact') {
    setPriority(95);  // Exact matches highest priority
  } else if (newType === 'regex') {
    setPriority(90);
  } else if (newType === 'fuzzy') {
    setPriority(85);
    setSimilarityThreshold(0.80);  // Default threshold
  } else if (newType === 'soundex') {
    setPriority(80);
  }

  // Clear test results
  setTestResult(null);
}
```

---

## Multi-Domain Examples

### Finance: Merchant Rule

```tsx
<RuleEditorDialog
  isOpen={true}
  mode="create"
  onSave={async (rule) => {
    return api.post('/api/normalization-rules', rule);
  }}
  onTest={async (pattern, type, threshold) => {
    return api.post('/api/normalization-rules/test', {
      pattern,
      rule_type: type,
      similarity_threshold: threshold
    });
  }}
  onCancel={() => setIsOpen(false)}
/>

// Example input:
// Pattern: "UBER.*"
// Replacement: "Uber"
// Type: Regex
// Priority: 90
```

### Healthcare: Provider Rule

```tsx
<RuleEditorDialog
  isOpen={true}
  mode="create"
  onSave={async (rule) => {
    return api.post('/api/provider-rules', rule);
  }}
  onTest={async (pattern, type, threshold) => {
    return api.post('/api/provider-rules/test', {
      pattern,
      rule_type: type,
      similarity_threshold: threshold
    });
  }}
  onCancel={() => setIsOpen(false)}
/>

// Example input:
// Pattern: "ST MARY'S.*"
// Replacement: "St. Mary's Hospital"
// Type: Regex
// Priority: 90
```

### Legal: Case Name Rule

```tsx
<RuleEditorDialog
  isOpen={true}
  mode="edit"
  rule={existingRule}
  onSave={async (rule) => {
    return api.patch(`/api/case-rules/${rule.rule_id}`, rule);
  }}
  onCancel={() => setIsOpen(false)}
/>
```

---

## Accessibility

### Form Validation

```tsx
<label htmlFor="pattern-input">
  Pattern *
  {validationErrors.pattern && (
    <span role="alert" className="error">
      {validationErrors.pattern}
    </span>
  )}
</label>
<input
  id="pattern-input"
  value={pattern}
  onChange={(e) => setPattern(e.target.value)}
  aria-invalid={!!validationErrors.pattern}
  aria-describedby={validationErrors.pattern ? "pattern-error" : undefined}
/>
```

### Keyboard Shortcuts

- `Ctrl+Enter` / `Cmd+Enter` - Save rule
- `Ctrl+T` / `Cmd+T` - Test rule
- `Esc` - Cancel / Close dialog

---

## Performance

### Debounced Test

```typescript
const debouncedTest = useMemo(
  () => debounce(handleTestRule, 500),
  [pattern, ruleType, similarityThreshold]
);

// Auto-test on pattern change (debounced)
useEffect(() => {
  if (pattern && replacement) {
    debouncedTest();
  }
}, [pattern, replacement, ruleType]);
```

---

## Related Components

- **MerchantRulesManager** - Parent component
- **ClusterViewer** - Auto-create rules from clusters
- **TransactionList** - View normalized results

---

## Summary

**RuleEditorDialog** provides:
- ✅ Create/edit normalization rule form
- ✅ 4 rule types (exact, regex, fuzzy, soundex)
- ✅ Real-time pattern validation
- ✅ Rule testing with preview
- ✅ Priority slider with recommendations
- ✅ Fuzzy threshold configuration
- ✅ Multi-domain support

**Universal Pattern:**
Applicable to any domain requiring user-defined text transformation rules.
