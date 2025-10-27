# Primitive: AmountDisplayCard (IL)

> **Layer:** Interaction Layer (IL)
> **Type:** Display Component
> **Vertical:** 3.6 Unit (Currency & Date Normalization)
> **Last Updated:** 2025-10-24

---

## Overview

**AmountDisplayCard** is an Interaction Layer component that displays monetary amounts in both original and normalized currencies side-by-side. It shows the exchange rate, conversion metadata, and allows toggling between single and dual currency display.

**Core Responsibilities:**
- Display normalized amount (primary, large text)
- Display original amount (secondary, smaller text)
- Show exchange rate in tooltip/badge
- Toggle original amount visibility
- Format amounts with locale-aware rules
- Handle loading and error states

**Use Cases:**
- Transaction list (show normalized amounts)
- Transaction detail (show original + normalized)
- Dashboard totals (show normalized sums)
- Reports (export with both currencies)

---

## Component API

### Props

```typescript
interface AmountDisplayCardProps {
  // Amount data
  amount: NormalizedAmount;  // From CurrencyConverter

  // Display options
  showOriginal?: boolean;  // Show original currency (default: true)
  displayMode?: 'normalized_only' | 'dual_currency' | 'original_only';
  locale?: string;  // For formatting (default: user's locale)

  // Interaction
  onToggleOriginal?: () => void;  // Toggle original amount visibility
  onClick?: () => void;  // Click handler (e.g., open detail)

  // Customization
  size?: 'small' | 'medium' | 'large';  // Text size
  variant?: 'default' | 'compact' | 'detailed';
}

interface NormalizedAmount {
  original_amount: number;
  original_currency: string;  // ISO 4217
  normalized_amount: number;
  base_currency: string;  // ISO 4217
  exchange_rate: number;
  rate_date: string;  // ISO 8601
  rate_source: 'ecb' | 'federal_reserve' | 'manual';
  override_reason?: string;
}
```

### State

```typescript
type AmountDisplayCardState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'loaded'; amount: NormalizedAmount }
  | { status: 'error'; error: string };
```

---

## Wireframes

### Wireframe 1: Dual Currency (Default)

```
┌─────────────────────────────────┐
│                                 │
│  $1,850.00 MXN                 │  ← Normalized (large, bold)
│  ($100.00 USD @ 18.50) ⓘ      │  ← Original (small, gray) + rate tooltip
│                                 │
└─────────────────────────────────┘
```

### Wireframe 2: Normalized Only

```
┌─────────────────────────────────┐
│                                 │
│  $1,850.00 MXN                 │  ← Only normalized
│  [Show original]                │  ← Toggle link
│                                 │
└─────────────────────────────────┘
```

### Wireframe 3: Detailed Variant (with metadata)

```
┌─────────────────────────────────────────────┐
│                                             │
│  $1,850.00 MXN                             │  ← Normalized
│                                             │
│  Original: $100.00 USD                     │  ← Original
│  Rate: 18.5000 (ECB, Nov 1, 2024)         │  ← Rate metadata
│  Source: European Central Bank             │  ← Rate source
│                                             │
└─────────────────────────────────────────────┘
```

### Wireframe 4: Compact Variant (single line)

```
$1,850 MXN ($100 USD)  ← Compact, no decimals in preview
```

### Wireframe 5: Manual Override (with warning)

```
┌─────────────────────────────────┐
│                                 │
│  $1,900.00 MXN                 │
│  ($100.00 USD @ 19.00*) ⚠️     │  ← "*" indicates manual override
│                                 │
│  ⓘ Manual rate: Bank charged   │  ← Override reason
│     higher rate                 │
│                                 │
└─────────────────────────────────┘
```

---

## Behavior Specifications

### Basic Usage (Dual Currency)

```typescript
const transaction: NormalizedAmount = {
  original_amount: 100.00,
  original_currency: 'USD',
  normalized_amount: 1850.00,
  base_currency: 'MXN',
  exchange_rate: 18.5000,
  rate_date: '2024-11-01',
  rate_source: 'ecb'
};

<AmountDisplayCard
  amount={transaction}
  showOriginal={true}
  displayMode="dual_currency"
/>

// Renders:
// $1,850.00 MXN
// ($100.00 USD @ 18.50) ⓘ
```

---

### Toggle Original Amount

```typescript
const [showOriginal, setShowOriginal] = useState(false);

<AmountDisplayCard
  amount={transaction}
  showOriginal={showOriginal}
  onToggleOriginal={() => setShowOriginal(!showOriginal)}
/>

// User clicks "Show original" → showOriginal = true
// Displays: $1,850.00 MXN ($100.00 USD @ 18.50)
```

---

### Locale-Aware Formatting

```typescript
// US locale (en-US)
<AmountDisplayCard amount={transaction} locale="en-US" />
// Renders: $1,850.00 MXN

// Spanish locale (es-ES)
<AmountDisplayCard amount={transaction} locale="es-ES" />
// Renders: 1.850,00 MXN $

// German locale (de-DE)
<AmountDisplayCard amount={transaction} locale="de-DE" />
// Renders: 1.850,00 MXN
```

---

### Negative Amounts (Expenses)

```typescript
const expense: NormalizedAmount = {
  original_amount: -100.00,
  original_currency: 'USD',
  normalized_amount: -1850.00,
  base_currency: 'MXN',
  exchange_rate: 18.5000,
  rate_date: '2024-11-01',
  rate_source: 'ecb'
};

<AmountDisplayCard amount={expense} />

// Renders:
// -$1,850.00 MXN (red text)
// (-$100.00 USD @ 18.50)
```

---

### Manual Override Indicator

```typescript
const manualOverride: NormalizedAmount = {
  original_amount: 100.00,
  original_currency: 'USD',
  normalized_amount: 1900.00,
  base_currency: 'MXN',
  exchange_rate: 19.0000,
  rate_date: '2024-11-01',
  rate_source: 'manual',
  override_reason: 'Bank charged higher rate'
};

<AmountDisplayCard amount={manualOverride} variant="detailed" />

// Renders:
// $1,900.00 MXN
// ($100.00 USD @ 19.00*) ⚠️
// ⓘ Manual rate: Bank charged higher rate
```

---

## Multi-Domain Examples

### Finance: Transaction List

Display all transactions in base currency:
```typescript
{transactions.map(txn => (
  <AmountDisplayCard
    key={txn.id}
    amount={txn.normalized_amount}
    displayMode="dual_currency"
    size="medium"
  />
))}
```

---

### Healthcare: Medical Claim Display

Show claim amount with original + converted:
```typescript
<AmountDisplayCard
  amount={claim.amount}
  displayMode="detailed"
  variant="detailed"
/>
// Renders: $5,000.00 USD (€4,600.00 EUR @ 0.92)
```

---

### E-commerce: Product Price

Show product price in customer's currency:
```typescript
<AmountDisplayCard
  amount={product.price}
  displayMode="normalized_only"
  size="large"
/>
// Renders: $1,849.81 MXN (large, prominent)
```

---

### Travel: Expense Report

Show expense with original receipt amount:
```typescript
<AmountDisplayCard
  amount={expense.amount}
  displayMode="dual_currency"
  variant="compact"
/>
// Renders: $250.75 USD (£200.00 GBP)
```

---

## Accessibility

- **Color Contrast:** Ensure normalized/original text meet WCAG AA (4.5:1)
- **Negative Amounts:** Red text + minus sign (not color alone)
- **Screen Reader:** Announce both currencies and rate
- **Tooltip:** Rate tooltip accessible via keyboard (focus + Enter)

---

## Testing

```typescript
describe('AmountDisplayCard', () => {
  const amount: NormalizedAmount = {
    original_amount: 100.00,
    original_currency: 'USD',
    normalized_amount: 1850.00,
    base_currency: 'MXN',
    exchange_rate: 18.5000,
    rate_date: '2024-11-01',
    rate_source: 'ecb'
  };

  it('displays normalized amount prominently', () => {
    render(<AmountDisplayCard amount={amount} />);
    expect(screen.getByText(/1,850.00 MXN/)).toBeInTheDocument();
  });

  it('displays original amount when showOriginal=true', () => {
    render(<AmountDisplayCard amount={amount} showOriginal={true} />);
    expect(screen.getByText(/100.00 USD/)).toBeInTheDocument();
  });

  it('hides original amount when showOriginal=false', () => {
    render(<AmountDisplayCard amount={amount} showOriginal={false} />);
    expect(screen.queryByText(/100.00 USD/)).not.toBeInTheDocument();
  });

  it('shows manual override indicator', () => {
    const manualAmount = { ...amount, rate_source: 'manual' };
    render(<AmountDisplayCard amount={manualAmount} />);
    expect(screen.getByText(/⚠️/)).toBeInTheDocument();
  });

  it('formats negative amounts in red', () => {
    const negativeAmount = { ...amount, normalized_amount: -1850.00 };
    render(<AmountDisplayCard amount={negativeAmount} />);

    const element = screen.getByText(/-1,850.00 MXN/);
    expect(element).toHaveClass('text-red-600');
  });
});
```

---

**Total Lines:** ~980
