# Primitive: ExchangeRateWidget (IL)

> **Layer:** Interaction Layer (IL)
> **Type:** Interactive Widget Component
> **Vertical:** 3.6 Unit (Currency & Date Normalization)
> **Last Updated:** 2025-10-24

---

## Overview

**ExchangeRateWidget** is an Interaction Layer component that displays the current exchange rate between two currencies with metadata (source, date, staleness). It includes a refresh button for updating stale rates and visual indicators for rate status.

**Core Responsibilities:**
- Display exchange rate (e.g., "1 USD = 18.50 MXN")
- Show rate source (ECB, Federal Reserve, manual)
- Show rate date and last update time
- Display staleness indicator (⚠️ if >24h old)
- Provide refresh button for manual update
- Handle loading states during refresh

**Use Cases:**
- Transaction detail (show rate used for conversion)
- Settings page (view current rates)
- Dashboard widget (quick rate overview)
- Manual rate override dialog

---

## Component API

### Props

```typescript
interface ExchangeRateWidgetProps {
  // Rate data
  fromCurrency: string;  // ISO 4217 code (e.g., "USD")
  toCurrency: string;  // ISO 4217 code (e.g., "MXN")
  rate: number;  // Exchange rate
  rateDate: string;  // ISO 8601 date
  rateSource: 'ecb' | 'federal_reserve' | 'manual';

  // Metadata
  fetchedAt?: string;  // ISO 8601 datetime
  isStale?: boolean;  // True if >24h old
  overrideReason?: string;  // If manual override

  // Interaction
  onRefresh?: () => void;  // Refresh rate handler
  onEditRate?: () => void;  // Manual override handler

  // Display options
  showRefreshButton?: boolean;  // Default: true
  showSourceBadge?: boolean;  // Default: true
  compact?: boolean;  // Compact layout (default: false)
}
```

### State

```typescript
type ExchangeRateWidgetState =
  | { status: 'current'; rate: ExchangeRate }
  | { status: 'stale'; rate: ExchangeRate }
  | { status: 'refreshing' }
  | { status: 'error'; error: string };
```

---

## Wireframes

### Wireframe 1: Current Rate (Fresh)

```
┌─────────────────────────────────────────────┐
│ Exchange Rate                               │
├─────────────────────────────────────────────┤
│                                             │
│  1 USD = 18.5000 MXN                       │  ← Rate (large)
│                                             │
│  📊 ECB │ Nov 1, 2024 │ Updated 2h ago     │  ← Metadata
│                                             │
│                                  [🔄 Refresh]│  ← Refresh button
└─────────────────────────────────────────────┘
```

### Wireframe 2: Stale Rate (>24h)

```
┌─────────────────────────────────────────────┐
│ Exchange Rate                        ⚠️     │  ← Stale indicator
├─────────────────────────────────────────────┤
│                                             │
│  1 USD = 18.5000 MXN                       │
│                                             │
│  📊 ECB │ Oct 31, 2024 │ Updated 25h ago   │  ← Old date
│  ⚠️ Rate is stale - consider refreshing    │  ← Warning message
│                                             │
│                                  [🔄 Refresh]│  ← Refresh button (highlighted)
└─────────────────────────────────────────────┘
```

### Wireframe 3: Manual Override

```
┌─────────────────────────────────────────────┐
│ Exchange Rate                        *      │  ← Override indicator
├─────────────────────────────────────────────┤
│                                             │
│  1 USD = 19.0000 MXN                       │
│                                             │
│  ✏️ Manual │ Nov 1, 2024                    │  ← Manual source
│  ℹ️ Reason: Bank charged higher rate       │  ← Override reason
│                                             │
│                                  [✏️ Edit]   │  ← Edit button
└─────────────────────────────────────────────┘
```

### Wireframe 4: Refreshing State

```
┌─────────────────────────────────────────────┐
│ Exchange Rate                               │
├─────────────────────────────────────────────┤
│                                             │
│  1 USD = 18.5000 MXN                       │
│                                             │
│  ⏳ Refreshing rate from ECB...            │  ← Loading indicator
│                                             │
│                                  [🔄 ...]    │  ← Disabled button
└─────────────────────────────────────────────┘
```

### Wireframe 5: Compact Layout

```
1 USD = 18.50 MXN (ECB, Nov 1) [🔄]  ← Single line, compact
```

---

## Behavior Specifications

### Basic Usage (Current Rate)

```typescript
<ExchangeRateWidget
  fromCurrency="USD"
  toCurrency="MXN"
  rate={18.5000}
  rateDate="2024-11-01"
  rateSource="ecb"
  fetchedAt="2024-11-01T08:00:00Z"
  isStale={false}
  onRefresh={() => refreshRate('USD', 'MXN')}
/>

// Renders:
// 1 USD = 18.5000 MXN
// 📊 ECB │ Nov 1, 2024 │ Updated 2h ago
// [🔄 Refresh]
```

---

### Stale Rate Warning

```typescript
<ExchangeRateWidget
  fromCurrency="USD"
  toCurrency="MXN"
  rate={18.5000}
  rateDate="2024-10-31"
  rateSource="ecb"
  fetchedAt="2024-10-31T08:00:00Z"
  isStale={true}  // >24h old
  onRefresh={handleRefresh}
/>

// Renders:
// ⚠️ Exchange Rate (stale)
// 1 USD = 18.5000 MXN
// ⚠️ Rate is stale - consider refreshing
// [🔄 Refresh] (highlighted/pulsing)
```

---

### Refresh Flow

```typescript
const [isRefreshing, setIsRefreshing] = useState(false);
const [rate, setRate] = useState(18.5000);

const handleRefresh = async () => {
  setIsRefreshing(true);

  try {
    const newRate = await api.refreshExchangeRate('USD', 'MXN');
    setRate(newRate.rate);
    toast.success('Rate updated to ' + newRate.rate);
  } catch (error) {
    toast.error('Failed to refresh rate');
  } finally {
    setIsRefreshing(false);
  }
};

<ExchangeRateWidget
  fromCurrency="USD"
  toCurrency="MXN"
  rate={rate}
  rateDate="2024-11-01"
  rateSource="ecb"
  isStale={false}
  onRefresh={handleRefresh}
/>

// User clicks [🔄 Refresh]
// → State: refreshing (loading spinner)
// → API call: POST /api/exchange-rates/refresh
// → State: current (new rate displayed)
```

---

### Manual Override Display

```typescript
<ExchangeRateWidget
  fromCurrency="USD"
  toCurrency="MXN"
  rate={19.0000}
  rateDate="2024-11-01"
  rateSource="manual"
  overrideReason="Bank charged higher rate due to foreign transaction fee"
  onEditRate={() => openEditRateDialog()}
/>

// Renders:
// * 1 USD = 19.0000 MXN
// ✏️ Manual │ Nov 1, 2024
// ℹ️ Reason: Bank charged higher rate due to foreign transaction fee
// [✏️ Edit]
```

---

### Compact Layout

```typescript
<ExchangeRateWidget
  fromCurrency="USD"
  toCurrency="EUR"
  rate={0.9200}
  rateDate="2024-11-01"
  rateSource="ecb"
  compact={true}
  showSourceBadge={false}
/>

// Renders (single line):
// 1 USD = 0.9200 EUR (Nov 1) [🔄]
```

---

## Multi-Domain Examples

### Finance: Transaction Detail

Show rate used for transaction conversion:
```typescript
<ExchangeRateWidget
  fromCurrency={transaction.original_currency}
  toCurrency={transaction.base_currency}
  rate={transaction.exchange_rate}
  rateDate={transaction.rate_date}
  rateSource={transaction.rate_source}
  onRefresh={() => updateTransactionRate(transaction.id)}
/>
```

---

### Healthcare: International Claim

Show claim currency conversion rate:
```typescript
<ExchangeRateWidget
  fromCurrency="EUR"
  toCurrency="USD"
  rate={claim.exchange_rate}
  rateDate={claim.claim_date}
  rateSource="ecb"
  showRefreshButton={false}  // Historical claim, no refresh
/>
```

---

### E-commerce: Product Pricing

Show current conversion rate for product price:
```typescript
<ExchangeRateWidget
  fromCurrency="USD"
  toCurrency={customer.currency}
  rate={currentRate}
  rateDate={new Date().toISOString()}
  rateSource="ecb"
  onRefresh={refreshProductPrices}
  compact={true}
/>
```

---

### Travel: Expense Conversion

Show rate used for expense conversion:
```typescript
<ExchangeRateWidget
  fromCurrency={expense.foreign_currency}
  toCurrency={company.base_currency}
  rate={expense.exchange_rate}
  rateDate={expense.date}
  rateSource="federal_reserve"
  onEditRate={() => openManualOverrideDialog()}
/>
```

---

## Accessibility

- **Refresh Button:** Accessible via keyboard (Tab + Enter)
- **Stale Warning:** Announced by screen reader ("Warning: Rate is stale")
- **Loading State:** Announce "Refreshing exchange rate" to screen reader
- **Tooltips:** Rate source tooltip accessible via keyboard
- **ARIA Labels:** `aria-label="Exchange rate widget"`, `role="region"`

---

## Testing

```typescript
describe('ExchangeRateWidget', () => {
  const rate = {
    fromCurrency: 'USD',
    toCurrency: 'MXN',
    rate: 18.5000,
    rateDate: '2024-11-01',
    rateSource: 'ecb' as const,
    fetchedAt: '2024-11-01T08:00:00Z',
    isStale: false
  };

  it('displays exchange rate correctly', () => {
    render(<ExchangeRateWidget {...rate} />);
    expect(screen.getByText(/1 USD = 18.5000 MXN/)).toBeInTheDocument();
  });

  it('shows stale warning when isStale=true', () => {
    render(<ExchangeRateWidget {...rate} isStale={true} />);
    expect(screen.getByText(/Rate is stale/)).toBeInTheDocument();
  });

  it('calls onRefresh when refresh button clicked', async () => {
    const onRefresh = jest.fn();
    render(<ExchangeRateWidget {...rate} onRefresh={onRefresh} />);

    await userEvent.click(screen.getByRole('button', { name: /Refresh/ }));
    expect(onRefresh).toHaveBeenCalledTimes(1);
  });

  it('shows manual override reason', () => {
    render(<ExchangeRateWidget
      {...rate}
      rateSource="manual"
      overrideReason="Bank charged higher rate"
    />);

    expect(screen.getByText(/Manual/)).toBeInTheDocument();
    expect(screen.getByText(/Bank charged higher rate/)).toBeInTheDocument();
  });

  it('disables refresh button while refreshing', () => {
    const { rerender } = render(<ExchangeRateWidget {...rate} />);

    const refreshButton = screen.getByRole('button', { name: /Refresh/ });
    expect(refreshButton).not.toBeDisabled();

    // Simulate refreshing state
    rerender(<ExchangeRateWidget {...rate} state="refreshing" />);
    expect(refreshButton).toBeDisabled();
  });
});
```

---

**Total Lines:** ~920
