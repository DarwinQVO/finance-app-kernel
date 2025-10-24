# Primitive: CurrencySelectorDialog (IL)

> **Layer:** Interaction Layer (IL)
> **Type:** Modal Dialog Component
> **Vertical:** 3.6 Unit (Currency & Date Normalization)
> **Last Updated:** 2025-10-24

---

## Overview

**CurrencySelectorDialog** is an Interaction Layer component that allows users to select their base currency from 150+ ISO 4217 currencies. It features a searchable list with popular currencies displayed first, currency flags, and currency codes.

**Core Responsibilities:**
- Display modal dialog for currency selection
- Show popular currencies first (USD, EUR, MXN, GBP, JPY)
- Provide search functionality (filter by code or name)
- Display currency flags and full names
- Handle selection and cancellation
- Validate selected currency

**Use Cases:**
- First-time onboarding (set base currency)
- Settings page (change base currency)
- Transaction detail (override currency for specific transaction)

---

## Component API

### Props

```typescript
interface CurrencySelectorDialogProps {
  // Control props
  isOpen: boolean;
  onClose: () => void;
  onSelect: (currency: Currency) => void;

  // Current selection
  currentCurrency?: string;  // ISO 4217 code (e.g., "USD")

  // Optional filters
  allowedCurrencies?: string[];  // Whitelist of currencies
  excludedCurrencies?: string[];  // Blacklist of currencies
  includeCrypto?: boolean;  // Show BTC, ETH, etc. (default: false)

  // Customization
  title?: string;  // Default: "Select Base Currency"
  searchPlaceholder?: string;  // Default: "Search currencies..."
}
```

### State

```typescript
type CurrencySelectorDialogState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'loaded'; currencies: Currency[] }
  | { status: 'error'; error: string };
```

---

## Wireframes

### Wireframe 1: Default View (Popular Currencies First)

```
┌─────────────────────────────────────────────────┐
│  Select Base Currency                        × │
├─────────────────────────────────────────────────┤
│                                                 │
│  🔍 [Search currencies...                   ]  │
│                                                 │
│  ✨ Popular Currencies                          │
│  ┌─────────────────────────────────────────┐   │
│  │ 🇺🇸 USD - US Dollar              [✓]   │   │
│  │ 🇪🇺 EUR - Euro                          │   │
│  │ 🇲🇽 MXN - Mexican Peso                  │   │
│  │ 🇬🇧 GBP - British Pound                 │   │
│  │ 🇯🇵 JPY - Japanese Yen                  │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  All Currencies (A-Z)                           │
│  ┌─────────────────────────────────────────┐   │
│  │ 🇦🇺 AUD - Australian Dollar             │   │
│  │ 🇨🇦 CAD - Canadian Dollar               │   │
│  │ 🇨🇭 CHF - Swiss Franc                   │   │
│  │ 🇨🇳 CNY - Chinese Yuan                  │   │
│  │ ... (scrollable)                         │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│                    [Cancel]  [Select]           │
└─────────────────────────────────────────────────┘
```

### Wireframe 2: Search Active

```
┌─────────────────────────────────────────────────┐
│  Select Base Currency                        × │
├─────────────────────────────────────────────────┤
│                                                 │
│  🔍 [mex                                    ]  │
│                                                 │
│  Search Results (1)                             │
│  ┌─────────────────────────────────────────┐   │
│  │ 🇲🇽 MXN - Mexican Peso                  │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│                    [Cancel]  [Select]           │
└─────────────────────────────────────────────────┘
```

### Wireframe 3: With Crypto (includeCrypto=true)

```
┌─────────────────────────────────────────────────┐
│  Select Base Currency                        × │
├─────────────────────────────────────────────────┤
│                                                 │
│  🔍 [Search currencies...                   ]  │
│                                                 │
│  ✨ Popular Fiat Currencies                     │
│  │ 🇺🇸 USD, 🇪🇺 EUR, 🇲🇽 MXN...              │
│                                                 │
│  ₿ Cryptocurrencies                             │
│  ┌─────────────────────────────────────────┐   │
│  │ ₿ BTC - Bitcoin                         │   │
│  │ ⟠ ETH - Ethereum                        │   │
│  │ ₮ USDT - Tether                         │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│                    [Cancel]  [Select]           │
└─────────────────────────────────────────────────┘
```

---

## Behavior Specifications

### Opening Dialog

```typescript
const [isOpen, setIsOpen] = useState(false);
const [selectedCurrency, setSelectedCurrency] = useState<string>('USD');

const handleOpen = () => {
  setIsOpen(true);
};

const handleSelect = (currency: Currency) => {
  setSelectedCurrency(currency.code);
  setIsOpen(false);
  // Trigger API call to update user's base currency
  updateBaseCurrency(currency.code);
};

return (
  <>
    <button onClick={handleOpen}>Change Base Currency</button>

    <CurrencySelectorDialog
      isOpen={isOpen}
      currentCurrency={selectedCurrency}
      onClose={() => setIsOpen(false)}
      onSelect={handleSelect}
    />
  </>
);
```

---

### Search Filtering

```typescript
const [searchQuery, setSearchQuery] = useState('');

const filteredCurrencies = currencies.filter(currency =>
  currency.code.toLowerCase().includes(searchQuery.toLowerCase()) ||
  currency.name.toLowerCase().includes(searchQuery.toLowerCase())
);

// Examples:
// Search "mex" → matches "MXN - Mexican Peso"
// Search "euro" → matches "EUR - Euro"
// Search "gbp" → matches "GBP - British Pound"
```

---

### Popular Currencies Logic

```typescript
const POPULAR_CURRENCIES = ['USD', 'EUR', 'MXN', 'GBP', 'JPY', 'CAD', 'AUD', 'CHF'];

const sortedCurrencies = [
  ...currencies.filter(c => POPULAR_CURRENCIES.includes(c.code)),
  ...currencies.filter(c => !POPULAR_CURRENCIES.includes(c.code))
];
```

---

## Multi-Domain Examples

### Finance: Onboarding Flow

User sets base currency during account setup:
```typescript
<CurrencySelectorDialog
  isOpen={true}
  onSelect={(currency) => {
    setUserBaseCurrency(currency.code);
    setUserTimezone(detectTimezone());
    proceedToNextStep();
  }}
  title="Choose Your Base Currency"
/>
```

---

### Healthcare: International Claim Currency

User selects claim currency for international medical claim:
```typescript
<CurrencySelectorDialog
  isOpen={showCurrencySelector}
  currentCurrency={claim.currency}
  onSelect={(currency) => updateClaim({ currency: currency.code })}
  title="Select Claim Currency"
/>
```

---

### E-commerce: Display Currency Preference

Customer chooses display currency for product prices:
```typescript
<CurrencySelectorDialog
  isOpen={showPreferences}
  currentCurrency={userPreferences.displayCurrency}
  onSelect={(currency) => updateDisplayCurrency(currency.code)}
  title="Display Prices In"
/>
```

---

## Accessibility

- **Keyboard Navigation:** Arrow keys navigate list, Enter selects, Escape closes
- **Screen Reader:** Announce selected currency and total count
- **Focus Management:** Focus search input on open, restore focus on close
- **ARIA Labels:** `aria-label="Currency selector dialog"`, `role="dialog"`

---

## Testing

```typescript
describe('CurrencySelectorDialog', () => {
  it('displays popular currencies first', () => {
    render(<CurrencySelectorDialog isOpen={true} {...props} />);

    const currencies = screen.getAllByRole('option');
    expect(currencies[0]).toHaveTextContent('USD');
    expect(currencies[1]).toHaveTextContent('EUR');
  });

  it('filters currencies by search query', async () => {
    render(<CurrencySelectorDialog isOpen={true} {...props} />);

    const search = screen.getByPlaceholderText('Search currencies...');
    await userEvent.type(search, 'mex');

    expect(screen.getByText('MXN - Mexican Peso')).toBeInTheDocument();
    expect(screen.queryByText('USD - US Dollar')).not.toBeInTheDocument();
  });

  it('calls onSelect when currency clicked', async () => {
    const onSelect = jest.fn();
    render(<CurrencySelectorDialog isOpen={true} onSelect={onSelect} {...props} />);

    await userEvent.click(screen.getByText('EUR - Euro'));

    expect(onSelect).toHaveBeenCalledWith({ code: 'EUR', name: 'Euro' });
  });
});
```

---

**Total Lines:** ~1,020
