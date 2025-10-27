# FXConversionCard (IL Component)

## Definition

**FXConversionCard** is a specialized widget for displaying foreign exchange (FX) conversion details within a relationship context. It shows exchange rates, converted amounts, market rate comparisons, and gain/loss calculations with visual indicators.

**Problem it solves:**
- Inconsistent display of FX conversion data across different interfaces
- No standard format for showing exchange rates, conversion amounts, and market comparisons
- Difficult to visualize FX gain/loss at a glance
- No unified pattern for showing rate sources (calculated, manual, market)
- Complex to implement market rate comparison and historical rate charts

**Solution:**
- Single component for all FX conversion display needs
- Consistent formatting (4 decimal places for rates, currency symbols, color-coded gains/losses)
- Built-in market rate comparison logic
- Visual indicators for rate sources
- Link to detailed FX conversion reports
- Optional historical rate chart view
- Responsive design (stacked layout on mobile)

---

## Interface Contract

```typescript
interface FXConversionCardProps {
  // Relationship context (from RelationshipPanel)
  relationship: {
    id: string;
    type: "fx_conversion";
    metadata: {
      fromCurrency: string;      // "USD", "MXN", "EUR"
      toCurrency: string;         // "USD", "MXN", "EUR"
      exchangeRate: number;       // 4 decimal places, e.g., 16.8450
      rateSource: "calculated" | "manual" | "market";
      fromAmount?: number;        // Original amount (optional)
      toAmount?: number;          // Converted amount (optional)
      conversionDate?: string;    // ISO date string
    };
  };

  // FX conversion details
  fxDetails: {
    fromCurrency: string;
    toCurrency: string;
    exchangeRate: number;         // Applied rate
    fromAmount: number;
    toAmount: number;
    rateSource: "calculated" | "manual" | "market";
    conversionDate: string;       // ISO date string
    reportId?: string;            // Link to FX conversion report
  };

  // Market rate comparison (optional)
  showMarketRateComparison?: boolean;
  marketRate?: {
    rate: number;                 // 4 decimal places
    source: string;               // "XE.com", "Bloomberg", "ECB"
    timestamp: string;            // ISO date string
    fxGainLoss?: {
      amount: number;             // Positive = gain, negative = loss
      percentage: number;         // Percentage difference
      currency: string;           // Currency of gain/loss
    };
  };

  // UI states
  loading?: boolean;
  error?: string | null;

  // Interaction
  onViewReport?: () => void;           // Navigate to FX conversion report
  onShowHistoricalChart?: () => void;  // Show historical rate chart (optional)
  showHistoricalChart?: boolean;       // Enable historical chart button

  // Customization
  theme?: "light" | "dark";
  compact?: boolean;                   // Compact layout for small screens
}
```

---

## Component Structure

```tsx
import React from 'react';

export const FXConversionCard: React.FC<FXConversionCardProps> = ({
  relationship,
  fxDetails,
  showMarketRateComparison = false,
  marketRate,
  loading = false,
  error = null,
  onViewReport,
  onShowHistoricalChart,
  showHistoricalChart = false,
  theme = "light",
  compact = false
}) => {
  // Loading state
  if (loading) {
    return (
      <div className={`fx-conversion-card theme-${theme} loading`}>
        <div className="skeleton skeleton-header"></div>
        <div className="skeleton skeleton-rate"></div>
        <div className="skeleton skeleton-amounts"></div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className={`fx-conversion-card theme-${theme} error`}>
        <div className="error-icon">⚠️</div>
        <div className="error-message">{error}</div>
      </div>
    );
  }

  // Format exchange rate (4 decimal places)
  const formattedRate = fxDetails.exchangeRate.toFixed(4);

  // Format amounts with currency symbols
  const formatCurrency = (amount: number, currency: string) => {
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: currency,
      minimumFractionDigits: 2,
      maximumFractionDigits: 2
    }).format(amount);
  };

  // Calculate FX gain/loss (if market rate available)
  const calculateGainLoss = () => {
    if (!showMarketRateComparison || !marketRate?.rate) return null;

    const appliedConversion = fxDetails.toAmount;
    const marketConversion = fxDetails.fromAmount * marketRate.rate;
    const difference = appliedConversion - marketConversion;
    const percentage = ((difference / marketConversion) * 100).toFixed(2);

    return {
      amount: difference,
      percentage: parseFloat(percentage),
      currency: fxDetails.toCurrency,
      isGain: difference > 0
    };
  };

  const gainLoss = calculateGainLoss();

  // Rate source indicator
  const getRateSourceIcon = (source: string) => {
    switch (source) {
      case "calculated": return "🧮";
      case "manual": return "✏️";
      case "market": return "📊";
      default: return "•";
    }
  };

  const getRateSourceLabel = (source: string) => {
    switch (source) {
      case "calculated": return "Calculated";
      case "manual": return "Manual entry";
      case "market": return "Market rate";
      default: return source;
    }
  };

  // Format date
  const formatDate = (dateString: string) => {
    const date = new Date(dateString);
    return new Intl.DateTimeFormat('en-US', {
      month: 'short',
      day: 'numeric',
      year: 'numeric'
    }).format(date);
  };

  return (
    <div className={`fx-conversion-card theme-${theme} ${compact ? 'compact' : ''}`}>
      {/* Header: Currency pair and date */}
      <div className="fx-card-header">
        <div className="currency-pair">
          <span className="from-currency">{fxDetails.fromCurrency}</span>
          <span className="arrow">→</span>
          <span className="to-currency">{fxDetails.toCurrency}</span>
        </div>
        <div className="conversion-date">{formatDate(fxDetails.conversionDate)}</div>
      </div>

      {/* Exchange rate */}
      <div className="exchange-rate-section">
        <div className="rate-label">Exchange Rate</div>
        <div className="rate-value">
          <span className="rate-number">{formattedRate}</span>
          <span className="rate-source">
            {getRateSourceIcon(fxDetails.rateSource)}
            {getRateSourceLabel(fxDetails.rateSource)}
          </span>
        </div>
      </div>

      {/* Conversion amounts */}
      <div className="conversion-amounts">
        <div className="amount-row from-amount">
          <span className="amount-label">From</span>
          <span className="amount-value">
            {formatCurrency(fxDetails.fromAmount, fxDetails.fromCurrency)}
          </span>
        </div>
        <div className="conversion-arrow">↓</div>
        <div className="amount-row to-amount">
          <span className="amount-label">To</span>
          <span className="amount-value">
            {formatCurrency(fxDetails.toAmount, fxDetails.toCurrency)}
          </span>
        </div>
      </div>

      {/* Market rate comparison (if enabled) */}
      {showMarketRateComparison && marketRate && (
        <div className="market-comparison">
          <div className="market-rate-row">
            <span className="market-rate-label">
              Market rate ({marketRate.source})
            </span>
            <span className="market-rate-value">{marketRate.rate.toFixed(4)}</span>
          </div>

          {/* FX gain/loss */}
          {gainLoss && (
            <div className={`fx-gain-loss ${gainLoss.isGain ? 'gain' : 'loss'}`}>
              <span className="gain-loss-icon">
                {gainLoss.isGain ? '↑' : '↓'}
              </span>
              <span className="gain-loss-label">
                {gainLoss.isGain ? 'FX Gain' : 'FX Loss'}
              </span>
              <span className="gain-loss-amount">
                {formatCurrency(Math.abs(gainLoss.amount), gainLoss.currency)}
              </span>
              <span className="gain-loss-percentage">
                ({gainLoss.isGain ? '+' : ''}{gainLoss.percentage}%)
              </span>
            </div>
          )}

          <div className="market-rate-timestamp">
            As of {formatDate(marketRate.timestamp)}
          </div>
        </div>
      )}

      {/* Actions */}
      <div className="fx-card-actions">
        {fxDetails.reportId && onViewReport && (
          <button
            className="action-button primary"
            onClick={onViewReport}
            aria-label="View FX conversion report"
          >
            <span className="button-icon">📄</span>
            View Report
          </button>
        )}

        {showHistoricalChart && onShowHistoricalChart && (
          <button
            className="action-button secondary"
            onClick={onShowHistoricalChart}
            aria-label="Show historical rate chart"
          >
            <span className="button-icon">📈</span>
            Historical Rates
          </button>
        )}
      </div>
    </div>
  );
};
```

---

## Styling

```css
.fx-conversion-card {
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: 12px;
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  transition: box-shadow 0.2s;
}

.fx-conversion-card:hover {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

/* Header */
.fx-card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-bottom: 12px;
  border-bottom: 1px solid var(--card-border);
}

.currency-pair {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 18px;
  font-weight: 600;
  color: var(--text-primary);
}

.from-currency,
.to-currency {
  font-family: 'SF Mono', 'Courier New', monospace;
  font-weight: 700;
}

.arrow {
  color: var(--text-secondary);
  font-size: 16px;
}

.conversion-date {
  font-size: 13px;
  color: var(--text-secondary);
}

/* Exchange rate section */
.exchange-rate-section {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.rate-label {
  font-size: 12px;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: var(--text-secondary);
}

.rate-value {
  display: flex;
  align-items: baseline;
  gap: 12px;
}

.rate-number {
  font-size: 32px;
  font-weight: 700;
  font-family: 'SF Mono', 'Courier New', monospace;
  color: var(--text-primary);
}

.rate-source {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  color: var(--text-secondary);
  padding: 4px 8px;
  background: var(--badge-bg);
  border-radius: 4px;
}

/* Conversion amounts */
.conversion-amounts {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 16px;
  background: var(--section-bg);
  border-radius: 8px;
}

.amount-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.amount-label {
  font-size: 13px;
  font-weight: 500;
  color: var(--text-secondary);
}

.amount-value {
  font-size: 18px;
  font-weight: 600;
  font-family: 'SF Mono', 'Courier New', monospace;
  color: var(--text-primary);
}

.conversion-arrow {
  align-self: center;
  color: var(--text-secondary);
  font-size: 20px;
  margin: 4px 0;
}

/* Market comparison */
.market-comparison {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 16px;
  background: var(--comparison-bg);
  border: 1px solid var(--comparison-border);
  border-radius: 8px;
}

.market-rate-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.market-rate-label {
  font-size: 13px;
  font-weight: 500;
  color: var(--text-secondary);
}

.market-rate-value {
  font-size: 16px;
  font-weight: 600;
  font-family: 'SF Mono', 'Courier New', monospace;
  color: var(--text-primary);
}

/* FX gain/loss */
.fx-gain-loss {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px;
  border-radius: 6px;
}

.fx-gain-loss.gain {
  background: rgba(16, 185, 129, 0.1);
  border: 1px solid rgba(16, 185, 129, 0.3);
}

.fx-gain-loss.loss {
  background: rgba(239, 68, 68, 0.1);
  border: 1px solid rgba(239, 68, 68, 0.3);
}

.gain-loss-icon {
  font-size: 18px;
}

.fx-gain-loss.gain .gain-loss-icon {
  color: #10b981;
}

.fx-gain-loss.loss .gain-loss-icon {
  color: #ef4444;
}

.gain-loss-label {
  font-size: 13px;
  font-weight: 500;
  color: var(--text-secondary);
  flex: 1;
}

.gain-loss-amount {
  font-size: 16px;
  font-weight: 700;
  font-family: 'SF Mono', 'Courier New', monospace;
}

.fx-gain-loss.gain .gain-loss-amount {
  color: #10b981;
}

.fx-gain-loss.loss .gain-loss-amount {
  color: #ef4444;
}

.gain-loss-percentage {
  font-size: 13px;
  font-weight: 600;
  margin-left: 4px;
}

.fx-gain-loss.gain .gain-loss-percentage {
  color: #059669;
}

.fx-gain-loss.loss .gain-loss-percentage {
  color: #dc2626;
}

.market-rate-timestamp {
  font-size: 11px;
  color: var(--text-tertiary);
  text-align: right;
}

/* Actions */
.fx-card-actions {
  display: flex;
  gap: 12px;
  padding-top: 8px;
  border-top: 1px solid var(--card-border);
}

.action-button {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 10px 16px;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.action-button.primary {
  background: var(--primary-color);
  color: white;
}

.action-button.primary:hover {
  background: var(--primary-color-hover);
  transform: translateY(-1px);
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.15);
}

.action-button.secondary {
  background: var(--secondary-bg);
  color: var(--text-primary);
  border: 1px solid var(--card-border);
}

.action-button.secondary:hover {
  background: var(--secondary-bg-hover);
  border-color: var(--primary-color);
}

.button-icon {
  font-size: 16px;
}

/* Compact layout */
.fx-conversion-card.compact {
  padding: 16px;
  gap: 12px;
}

.fx-conversion-card.compact .rate-number {
  font-size: 24px;
}

.fx-conversion-card.compact .amount-value {
  font-size: 16px;
}

.fx-conversion-card.compact .fx-card-actions {
  flex-direction: column;
}

/* Loading skeleton */
.skeleton {
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: loading 1.5s infinite;
  border-radius: 4px;
}

.skeleton-header {
  height: 24px;
  width: 60%;
  margin-bottom: 16px;
}

.skeleton-rate {
  height: 40px;
  width: 80%;
  margin-bottom: 16px;
}

.skeleton-amounts {
  height: 80px;
  width: 100%;
}

@keyframes loading {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

/* Theme: Light */
.theme-light {
  --card-bg: #ffffff;
  --card-border: #e5e7eb;
  --text-primary: #111827;
  --text-secondary: #6b7280;
  --text-tertiary: #9ca3af;
  --section-bg: #f9fafb;
  --badge-bg: #f3f4f6;
  --comparison-bg: #fffbeb;
  --comparison-border: #fde68a;
  --primary-color: #3b82f6;
  --primary-color-hover: #2563eb;
  --secondary-bg: #f9fafb;
  --secondary-bg-hover: #f3f4f6;
}

/* Theme: Dark */
.theme-dark {
  --card-bg: #1f2937;
  --card-border: #374151;
  --text-primary: #f3f4f6;
  --text-secondary: #9ca3af;
  --text-tertiary: #6b7280;
  --section-bg: #111827;
  --badge-bg: #374151;
  --comparison-bg: #422006;
  --comparison-border: #78350f;
  --primary-color: #3b82f6;
  --primary-color-hover: #2563eb;
  --secondary-bg: #374151;
  --secondary-bg-hover: #4b5563;
}

/* Mobile responsive (< 640px) */
@media (max-width: 640px) {
  .fx-conversion-card {
    padding: 16px;
    gap: 12px;
  }

  .fx-card-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }

  .currency-pair {
    font-size: 16px;
  }

  .rate-number {
    font-size: 24px !important;
  }

  .rate-value {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }

  .amount-value {
    font-size: 16px !important;
  }

  .fx-card-actions {
    flex-direction: column;
  }

  .action-button {
    width: 100%;
    justify-content: center;
  }
}
```

---

## Visual Design (ASCII Wireframes)

### Basic State (No Market Rate Comparison)

```
┌─────────────────────────────────────────────────┐
│  USD → MXN                     Jan 15, 2025     │
├─────────────────────────────────────────────────┤
│                                                  │
│  EXCHANGE RATE                                   │
│  16.8450  🧮 Calculated                          │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  From             USD 1,000.00           │   │
│  │         ↓                                │   │
│  │  To               MXN 16,845.00          │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
├─────────────────────────────────────────────────┤
│  📄 View Report    📈 Historical Rates          │
└─────────────────────────────────────────────────┘
```

### With Market Rate Comparison (Gain)

```
┌─────────────────────────────────────────────────┐
│  USD → MXN                     Jan 15, 2025     │
├─────────────────────────────────────────────────┤
│                                                  │
│  EXCHANGE RATE                                   │
│  16.8450  ✏️ Manual entry                        │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  From             USD 1,000.00           │   │
│  │         ↓                                │   │
│  │  To               MXN 16,845.00          │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  Market rate (XE.com)        16.7980     │   │
│  │                                          │   │
│  │  ┌────────────────────────────────────┐ │   │
│  │  │ ↑ FX Gain    MXN 47.00  (+0.28%)   │ │   │
│  │  └────────────────────────────────────┘ │   │
│  │                                          │   │
│  │  As of Jan 15, 2025                     │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
├─────────────────────────────────────────────────┤
│  📄 View Report    📈 Historical Rates          │
└─────────────────────────────────────────────────┘
```

### With Market Rate Comparison (Loss)

```
┌─────────────────────────────────────────────────┐
│  EUR → USD                     Mar 20, 2025     │
├─────────────────────────────────────────────────┤
│                                                  │
│  EXCHANGE RATE                                   │
│  1.0520  📊 Market rate                          │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  From             EUR 5,000.00           │   │
│  │         ↓                                │   │
│  │  To               USD 5,260.00           │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  Market rate (Bloomberg)     1.0580      │   │
│  │                                          │   │
│  │  ┌────────────────────────────────────┐ │   │
│  │  │ ↓ FX Loss    USD 30.00  (-0.57%)   │ │   │
│  │  └────────────────────────────────────┘ │   │
│  │                                          │   │
│  │  As of Mar 20, 2025                     │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
├─────────────────────────────────────────────────┤
│  📄 View Report                                 │
└─────────────────────────────────────────────────┘
```

### Loading State

```
┌─────────────────────────────────────────────────┐
│  ████████                                       │
├─────────────────────────────────────────────────┤
│                                                  │
│  ████████████████                               │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  ██████████████████                      │   │
│  │  ██████████████████                      │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
└─────────────────────────────────────────────────┘
```

### Mobile Layout (Stacked)

```
┌──────────────────────────┐
│  USD → MXN               │
│  Jan 15, 2025            │
├──────────────────────────┤
│                          │
│  EXCHANGE RATE           │
│  16.8450                 │
│  🧮 Calculated           │
│                          │
│  ┌────────────────────┐  │
│  │  From              │  │
│  │  USD 1,000.00      │  │
│  │        ↓           │  │
│  │  To                │  │
│  │  MXN 16,845.00     │  │
│  └────────────────────┘  │
│                          │
│  ┌────────────────────┐  │
│  │ Market (XE.com)    │  │
│  │ 16.7980            │  │
│  │                    │  │
│  │ ┌────────────────┐ │  │
│  │ │ ↑ FX Gain      │ │  │
│  │ │ MXN 47.00      │ │  │
│  │ │ (+0.28%)       │ │  │
│  │ └────────────────┘ │  │
│  └────────────────────┘  │
│                          │
├──────────────────────────┤
│  📄 View Report          │
│  📈 Historical Rates     │
└──────────────────────────┘
```

---

## Multi-Domain Applicability

The FXConversionCard follows a **Currency/Unit Conversion Display** pattern that applies across many domains where unit conversion, rate comparison, and gain/loss tracking are needed.

### 1. Finance Domain (FX Trading)

**Use case:** Display currency conversion for international wire transfer

```tsx
<FXConversionCard
  relationship={{
    id: "rel_001",
    type: "fx_conversion",
    metadata: {
      fromCurrency: "USD",
      toCurrency: "MXN",
      exchangeRate: 16.8450,
      rateSource: "market"
    }
  }}
  fxDetails={{
    fromCurrency: "USD",
    toCurrency: "MXN",
    exchangeRate: 16.8450,
    fromAmount: 10000.00,
    toAmount: 168450.00,
    rateSource: "market",
    conversionDate: "2025-01-15T10:30:00Z",
    reportId: "fx_report_001"
  }}
  showMarketRateComparison={true}
  marketRate={{
    rate: 16.7980,
    source: "XE.com",
    timestamp: "2025-01-15T10:30:00Z",
    fxGainLoss: {
      amount: 470.00,
      percentage: 0.28,
      currency: "MXN"
    }
  }}
  onViewReport={() => navigateTo('/fx-reports/fx_report_001')}
  onShowHistoricalChart={() => openChartModal('USD-MXN')}
  showHistoricalChart={true}
/>
```

**Business impact:** Traders can quickly assess if they got a favorable rate and track FX gains/losses for tax purposes.

---

### 2. Healthcare Domain (International Billing)

**Use case:** Display currency conversion for international patient billing (medical tourism)

```tsx
<FXConversionCard
  relationship={{
    id: "rel_health_001",
    type: "fx_conversion",
    metadata: {
      fromCurrency: "EUR",
      toCurrency: "USD",
      exchangeRate: 1.0850,
      rateSource: "calculated"
    }
  }}
  fxDetails={{
    fromCurrency: "EUR",
    toCurrency: "USD",
    exchangeRate: 1.0850,
    fromAmount: 12000.00,  // Surgery cost in EUR
    toAmount: 13020.00,    // Billed in USD
    rateSource: "calculated",
    conversionDate: "2025-03-10T08:00:00Z",
    reportId: "billing_001"
  }}
  showMarketRateComparison={true}
  marketRate={{
    rate: 1.0920,
    source: "ECB",
    timestamp: "2025-03-10T08:00:00Z",
    fxGainLoss: {
      amount: -84.00,
      percentage: -0.64,
      currency: "USD"
    }
  }}
  onViewReport={() => viewBillingDetails('billing_001')}
  theme="light"
/>
```

**Domain adaptation:**
- Replace "FX Gain/Loss" with "Rate Variance"
- Add context: "Patient billed in USD, hospital receives EUR"
- Link to insurance claim report

---

### 3. Research Domain (RSRCH - Utilitario) - International Scraping Service Payments

**Use case:** Display conversion for international web scraping service (billed in EUR, paid in USD)

```tsx
<FXConversionCard
  relationship={{
    id: "rel_scraping_service_001",
    type: "fx_conversion",
    metadata: {
      fromCurrency: "EUR",
      toCurrency: "USD",
      exchangeRate: 1.0650,
      rateSource: "manual"
    }
  }}
  fxDetails={{
    fromCurrency: "EUR",
    toCurrency: "USD",
    exchangeRate: 1.0650,
    fromAmount: 50000.00,  // Grant amount in EUR
    toAmount: 53250.00,    // Available in USD
    rateSource: "manual",
    conversionDate: "2024-09-01T00:00:00Z",
    reportId: "grant_erc_2024_001"
  }}
  showMarketRateComparison={true}
  marketRate={{
    rate: 1.0720,
    source: "ECB",
    timestamp: "2024-09-01T00:00:00Z",
    fxGainLoss: {
      amount: -350.00,
      percentage: -0.65,
      currency: "USD"
    }
  }}
  onViewReport={() => viewGrantReport('grant_erc_2024_001')}
  theme="light"
/>
```

**Domain adaptation:**
- Label: "Grant Conversion" instead of "FX Conversion"
- Add context: "European Research Council Grant 2024"
- Show remaining budget in both currencies
- Link to grant expenditure report

---

### 4. Manufacturing Domain (Raw Material Cost Conversion)

**Use case:** Display currency conversion for imported raw materials (quoted in CNY, purchased in USD)

```tsx
<FXConversionCard
  relationship={{
    id: "rel_purchase_001",
    type: "fx_conversion",
    metadata: {
      fromCurrency: "CNY",
      toCurrency: "USD",
      exchangeRate: 0.1385,
      rateSource: "market"
    }
  }}
  fxDetails={{
    fromCurrency: "CNY",
    toCurrency: "USD",
    exchangeRate: 0.1385,
    fromAmount: 500000.00,  // Steel cost in CNY
    toAmount: 69250.00,     // Cost in USD
    rateSource: "market",
    conversionDate: "2025-02-20T06:00:00Z",
    reportId: "po_2025_0220"
  }}
  showMarketRateComparison={true}
  marketRate={{
    rate: 0.1390,
    source: "Bloomberg",
    timestamp: "2025-02-20T06:00:00Z",
    fxGainLoss: {
      amount: -250.00,
      percentage: -0.36,
      currency: "USD"
    }
  }}
  onViewReport={() => viewPurchaseOrder('po_2025_0220')}
  onShowHistoricalChart={() => openChartModal('CNY-USD')}
  showHistoricalChart={true}
/>
```

**Domain adaptation:**
- Label: "Material Cost Conversion"
- Add context: "Purchase Order #2025-0220 - Steel Sheet Import"
- Show impact on unit cost and margin
- Link to purchase order details

---

### 5. E-commerce Domain (International Sales)

**Use case:** Display currency conversion for international customer order

```tsx
<FXConversionCard
  relationship={{
    id: "rel_order_001",
    type: "fx_conversion",
    metadata: {
      fromCurrency: "GBP",
      toCurrency: "USD",
      exchangeRate: 1.2650,
      rateSource: "calculated"
    }
  }}
  fxDetails={{
    fromCurrency: "GBP",
    toCurrency: "USD",
    exchangeRate: 1.2650,
    fromAmount: 450.00,    // Customer paid in GBP
    toAmount: 569.25,      // Received in USD
    rateSource: "calculated",
    conversionDate: "2025-04-05T15:22:00Z",
    reportId: "order_20250405_001"
  }}
  showMarketRateComparison={true}
  marketRate={{
    rate: 1.2680,
    source: "Stripe",
    timestamp: "2025-04-05T15:22:00Z",
    fxGainLoss: {
      amount: -1.35,
      percentage: -0.24,
      currency: "USD"
    }
  }}
  onViewReport={() => viewOrderDetails('order_20250405_001')}
  compact={true}
/>
```

**Domain adaptation:**
- Label: "Payment Conversion"
- Add context: "Order #20250405-001"
- Show customer display currency vs. settlement currency
- Highlight payment processor fees

---

### 6. Energy Domain (Commodity Price Conversion)

**Use case:** Display unit conversion for oil barrel pricing (quoted in USD, purchased in EUR)

```tsx
<FXConversionCard
  relationship={{
    id: "rel_energy_001",
    type: "fx_conversion",
    metadata: {
      fromCurrency: "USD",
      toCurrency: "EUR",
      exchangeRate: 0.9220,
      rateSource: "market"
    }
  }}
  fxDetails={{
    fromCurrency: "USD",
    toCurrency: "EUR",
    exchangeRate: 0.9220,
    fromAmount: 850000.00,  // 10,000 barrels @ $85/barrel
    toAmount: 783700.00,    // Cost in EUR
    rateSource: "market",
    conversionDate: "2025-05-12T09:00:00Z",
    reportId: "energy_purchase_001"
  }}
  showMarketRateComparison={true}
  marketRate={{
    rate: 0.9250,
    source: "Reuters",
    timestamp: "2025-05-12T09:00:00Z",
    fxGainLoss: {
      amount: -2550.00,
      percentage: -0.32,
      currency: "EUR"
    }
  }}
  onViewReport={() => viewEnergyPurchase('energy_purchase_001')}
  onShowHistoricalChart={() => openChartModal('USD-EUR')}
  showHistoricalChart={true}
/>
```

**Domain adaptation:**
- Label: "Commodity Price Conversion"
- Add context: "10,000 barrels @ $85/barrel"
- Show per-unit cost in local currency
- Link to commodity futures chart

---

### 7. Real Estate Domain (International Property Transaction)

**Use case:** Display currency conversion for overseas property purchase

```tsx
<FXConversionCard
  relationship={{
    id: "rel_property_001",
    type: "fx_conversion",
    metadata: {
      fromCurrency: "THB",
      toCurrency: "USD",
      exchangeRate: 0.0285,
      rateSource: "manual"
    }
  }}
  fxDetails={{
    fromCurrency: "THB",
    toCurrency: "USD",
    exchangeRate: 0.0285,
    fromAmount: 12500000.00,  // Property price in Thai Baht
    toAmount: 356250.00,       // Price in USD
    rateSource: "manual",
    conversionDate: "2025-06-01T00:00:00Z",
    reportId: "property_bangkok_001"
  }}
  showMarketRateComparison={true}
  marketRate={{
    rate: 0.0290,
    source: "Bank of Thailand",
    timestamp: "2025-06-01T00:00:00Z",
    fxGainLoss: {
      amount: -6250.00,
      percentage: -1.72,
      currency: "USD"
    }
  }}
  onViewReport={() => viewPropertyTransaction('property_bangkok_001')}
/>
```

**Domain adaptation:**
- Label: "Property Price Conversion"
- Add context: "Condo in Bangkok - Sukhumvit District"
- Show escrow account details
- Link to transaction closing documents

---

## States and Variants

### 1. Basic State (No Market Comparison)

Used when market rate data is unavailable or not required.

```tsx
<FXConversionCard
  relationship={relationship}
  fxDetails={{
    fromCurrency: "USD",
    toCurrency: "MXN",
    exchangeRate: 16.8450,
    fromAmount: 1000.00,
    toAmount: 16845.00,
    rateSource: "calculated",
    conversionDate: "2025-01-15T10:30:00Z"
  }}
  showMarketRateComparison={false}
  onViewReport={() => navigateTo('/fx-report')}
/>
```

**Visual:** No market comparison section, simple conversion display.

---

### 2. Market Comparison State (With Gain)

Used when market rate is available and shows favorable exchange.

```tsx
<FXConversionCard
  relationship={relationship}
  fxDetails={fxDetails}
  showMarketRateComparison={true}
  marketRate={{
    rate: 16.7980,
    source: "XE.com",
    timestamp: "2025-01-15T10:30:00Z",
    fxGainLoss: {
      amount: 470.00,      // Positive = gain
      percentage: 0.28,
      currency: "MXN"
    }
  }}
/>
```

**Visual:** Green gain indicator with upward arrow.

---

### 3. Market Comparison State (With Loss)

Used when market rate shows unfavorable exchange.

```tsx
<FXConversionCard
  relationship={relationship}
  fxDetails={fxDetails}
  showMarketRateComparison={true}
  marketRate={{
    rate: 16.9200,
    source: "XE.com",
    timestamp: "2025-01-15T10:30:00Z",
    fxGainLoss: {
      amount: -750.00,     // Negative = loss
      percentage: -0.44,
      currency: "MXN"
    }
  }}
/>
```

**Visual:** Red loss indicator with downward arrow.

---

### 4. Loading State

Used during async data fetching.

```tsx
<FXConversionCard
  relationship={relationship}
  fxDetails={fxDetails}
  loading={true}
/>
```

**Visual:** Skeleton loaders for header, rate, and amounts.

---

### 5. Error State

Used when data fetch fails.

```tsx
<FXConversionCard
  relationship={relationship}
  fxDetails={fxDetails}
  error="Failed to fetch market rate data"
/>
```

**Visual:** Error icon with message.

---

### 6. Compact State (Mobile)

Used on small screens or embedded contexts.

```tsx
<FXConversionCard
  relationship={relationship}
  fxDetails={fxDetails}
  compact={true}
/>
```

**Visual:** Reduced padding, stacked buttons, smaller font sizes.

---

## Accessibility

### ARIA Labels

```tsx
<div
  className="fx-conversion-card"
  role="region"
  aria-label={`FX conversion: ${fxDetails.fromCurrency} to ${fxDetails.toCurrency}`}
>
  <div className="exchange-rate-section" aria-label="Exchange rate information">
    <div className="rate-value" aria-label={`Exchange rate: ${formattedRate}`}>
      {/* Rate display */}
    </div>
  </div>

  <div className="conversion-amounts" aria-label="Conversion amounts">
    {/* Amounts */}
  </div>

  {gainLoss && (
    <div
      className="fx-gain-loss"
      role="status"
      aria-live="polite"
      aria-label={`${gainLoss.isGain ? 'FX Gain' : 'FX Loss'}: ${formatCurrency(Math.abs(gainLoss.amount), gainLoss.currency)}, ${gainLoss.percentage}%`}
    >
      {/* Gain/loss display */}
    </div>
  )}
</div>
```

### Keyboard Navigation

```tsx
// Focus management for interactive elements
const handleKeyPress = (e: React.KeyboardEvent, action: () => void) => {
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault();
    action();
  }
};

// Action buttons
<button
  className="action-button primary"
  onClick={onViewReport}
  onKeyPress={(e) => handleKeyPress(e, onViewReport)}
  aria-label="View detailed FX conversion report"
  tabIndex={0}
>
  {/* Button content */}
</button>
```

### Screen Reader Support

```tsx
// Announce rate source
<span className="rate-source" aria-label={`Rate source: ${getRateSourceLabel(fxDetails.rateSource)}`}>
  {getRateSourceIcon(fxDetails.rateSource)}
  {getRateSourceLabel(fxDetails.rateSource)}
</span>

// Announce gain/loss with context
{gainLoss && (
  <div role="status" aria-live="polite">
    <span className="sr-only">
      Compared to market rate, this conversion resulted in
      {gainLoss.isGain ? 'a gain' : 'a loss'} of
      {formatCurrency(Math.abs(gainLoss.amount), gainLoss.currency)},
      which is {Math.abs(gainLoss.percentage)}%
      {gainLoss.isGain ? 'above' : 'below'} market rate.
    </span>
    {/* Visual display */}
  </div>
)}
```

### Color Contrast

- **Gain indicator (green):** `#10b981` on white background = 3.5:1 (AA for large text)
- **Loss indicator (red):** `#ef4444` on white background = 3.8:1 (AA for large text)
- **Primary text:** `#111827` on white = 16:1 (AAA)
- **Secondary text:** `#6b7280` on white = 4.7:1 (AA)

### Focus Indicators

```css
.action-button:focus {
  outline: 2px solid var(--primary-color);
  outline-offset: 2px;
}

.action-button:focus-visible {
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.3);
}
```

---

## Mobile Responsive Design

### Breakpoint Strategy

```css
/* Desktop (default): 640px+ */
.fx-conversion-card {
  padding: 20px;
  gap: 16px;
}

/* Tablet: 480px - 639px */
@media (max-width: 639px) {
  .fx-conversion-card {
    padding: 16px;
    gap: 12px;
  }

  .rate-number {
    font-size: 28px;
  }

  .amount-value {
    font-size: 16px;
  }
}

/* Mobile: < 480px */
@media (max-width: 479px) {
  .fx-conversion-card {
    padding: 12px;
    gap: 12px;
  }

  .fx-card-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }

  .currency-pair {
    font-size: 16px;
  }

  .rate-number {
    font-size: 24px;
  }

  .rate-value {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }

  .amount-value {
    font-size: 15px;
  }

  .fx-card-actions {
    flex-direction: column;
    gap: 8px;
  }

  .action-button {
    width: 100%;
    justify-content: center;
  }
}
```

### Touch Target Sizing

All interactive elements meet WCAG 2.2 Level AA requirements (minimum 44x44px touch target):

```css
.action-button {
  min-height: 44px;
  padding: 10px 16px;
  touch-action: manipulation; /* Prevent double-tap zoom */
}

/* Increase touch area on mobile */
@media (max-width: 479px) {
  .action-button {
    min-height: 48px;
    padding: 12px 16px;
  }
}
```

---

## Testing Guidance

### Unit Tests

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { FXConversionCard } from './FXConversionCard';

describe('FXConversionCard', () => {
  const mockRelationship = {
    id: 'rel_001',
    type: 'fx_conversion' as const,
    metadata: {
      fromCurrency: 'USD',
      toCurrency: 'MXN',
      exchangeRate: 16.8450,
      rateSource: 'calculated' as const
    }
  };

  const mockFxDetails = {
    fromCurrency: 'USD',
    toCurrency: 'MXN',
    exchangeRate: 16.8450,
    fromAmount: 1000.00,
    toAmount: 16845.00,
    rateSource: 'calculated' as const,
    conversionDate: '2025-01-15T10:30:00Z',
    reportId: 'fx_report_001'
  };

  test('renders exchange rate with 4 decimal places', () => {
    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
      />
    );

    expect(screen.getByText('16.8450')).toBeInTheDocument();
  });

  test('displays currency pair correctly', () => {
    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
      />
    );

    expect(screen.getByText('USD')).toBeInTheDocument();
    expect(screen.getByText('MXN')).toBeInTheDocument();
  });

  test('formats amounts with currency symbols', () => {
    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
      />
    );

    expect(screen.getByText('$1,000.00')).toBeInTheDocument();
    expect(screen.getByText('MX$16,845.00')).toBeInTheDocument();
  });

  test('shows FX gain with green indicator', () => {
    const marketRate = {
      rate: 16.7980,
      source: 'XE.com',
      timestamp: '2025-01-15T10:30:00Z',
      fxGainLoss: {
        amount: 470.00,
        percentage: 0.28,
        currency: 'MXN'
      }
    };

    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
        showMarketRateComparison={true}
        marketRate={marketRate}
      />
    );

    const gainElement = screen.getByText('FX Gain');
    expect(gainElement).toBeInTheDocument();
    expect(gainElement.closest('.fx-gain-loss')).toHaveClass('gain');
  });

  test('shows FX loss with red indicator', () => {
    const marketRate = {
      rate: 16.9200,
      source: 'XE.com',
      timestamp: '2025-01-15T10:30:00Z',
      fxGainLoss: {
        amount: -750.00,
        percentage: -0.44,
        currency: 'MXN'
      }
    };

    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
        showMarketRateComparison={true}
        marketRate={marketRate}
      />
    );

    const lossElement = screen.getByText('FX Loss');
    expect(lossElement).toBeInTheDocument();
    expect(lossElement.closest('.fx-gain-loss')).toHaveClass('loss');
  });

  test('calls onViewReport when button clicked', () => {
    const mockOnViewReport = jest.fn();

    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
        onViewReport={mockOnViewReport}
      />
    );

    const button = screen.getByText('View Report');
    fireEvent.click(button);

    expect(mockOnViewReport).toHaveBeenCalledTimes(1);
  });

  test('shows loading skeleton when loading=true', () => {
    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
        loading={true}
      />
    );

    expect(screen.getByTestId('skeleton-header')).toBeInTheDocument();
  });

  test('shows error message when error provided', () => {
    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
        error="Failed to fetch market rate"
      />
    );

    expect(screen.getByText('Failed to fetch market rate')).toBeInTheDocument();
  });

  test('displays rate source icon and label', () => {
    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={{...mockFxDetails, rateSource: 'manual'}}
      />
    );

    expect(screen.getByText('✏️')).toBeInTheDocument();
    expect(screen.getByText('Manual entry')).toBeInTheDocument();
  });
});
```

### Integration Tests

```typescript
describe('FXConversionCard Integration', () => {
  test('fetches and displays market rate comparison', async () => {
    const mockFetchMarketRate = jest.fn().mockResolvedValue({
      rate: 16.7980,
      source: 'XE.com',
      timestamp: '2025-01-15T10:30:00Z'
    });

    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
        showMarketRateComparison={true}
        marketRate={await mockFetchMarketRate()}
      />
    );

    await waitFor(() => {
      expect(screen.getByText('16.7980')).toBeInTheDocument();
      expect(screen.getByText('XE.com')).toBeInTheDocument();
    });
  });

  test('navigates to FX report when link clicked', async () => {
    const mockNavigate = jest.fn();

    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
        onViewReport={() => mockNavigate('/fx-reports/fx_report_001')}
      />
    );

    fireEvent.click(screen.getByText('View Report'));

    expect(mockNavigate).toHaveBeenCalledWith('/fx-reports/fx_report_001');
  });
});
```

### Accessibility Tests

```typescript
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('FXConversionCard Accessibility', () => {
  test('should have no accessibility violations', async () => {
    const { container } = render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
      />
    );

    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  test('keyboard navigation works correctly', () => {
    const mockOnViewReport = jest.fn();

    render(
      <FXConversionCard
        relationship={mockRelationship}
        fxDetails={mockFxDetails}
        onViewReport={mockOnViewReport}
      />
    );

    const button = screen.getByText('View Report');
    button.focus();
    fireEvent.keyPress(button, { key: 'Enter', code: 13 });

    expect(mockOnViewReport).toHaveBeenCalled();
  });
});
```

### Visual Regression Tests

```typescript
import { percySnapshot } from '@percy/playwright';

test('FXConversionCard visual regression', async ({ page }) => {
  await page.goto('/components/fx-conversion-card');

  // Basic state
  await percySnapshot(page, 'FXConversionCard - Basic State');

  // With market comparison (gain)
  await page.click('[data-testid="toggle-market-comparison"]');
  await page.click('[data-testid="simulate-gain"]');
  await percySnapshot(page, 'FXConversionCard - Market Comparison (Gain)');

  // With market comparison (loss)
  await page.click('[data-testid="simulate-loss"]');
  await percySnapshot(page, 'FXConversionCard - Market Comparison (Loss)');

  // Loading state
  await page.click('[data-testid="toggle-loading"]');
  await percySnapshot(page, 'FXConversionCard - Loading State');

  // Mobile view
  await page.setViewportSize({ width: 375, height: 667 });
  await percySnapshot(page, 'FXConversionCard - Mobile View');
});
```

---

## Implementation Notes

### 1. Exchange Rate Precision

Always display exchange rates with **4 decimal places** for accuracy:

```typescript
const formattedRate = fxDetails.exchangeRate.toFixed(4);
```

### 2. Gain/Loss Calculation

Calculate FX gain/loss by comparing applied rate to market rate:

```typescript
const calculateGainLoss = () => {
  if (!marketRate?.rate) return null;

  const appliedConversion = fxDetails.toAmount;
  const marketConversion = fxDetails.fromAmount * marketRate.rate;
  const difference = appliedConversion - marketConversion;
  const percentage = ((difference / marketConversion) * 100).toFixed(2);

  return {
    amount: difference,
    percentage: parseFloat(percentage),
    currency: fxDetails.toCurrency,
    isGain: difference > 0
  };
};
```

### 3. Rate Source Indicators

Provide visual cues for rate source trustworthiness:

- **Calculated (🧮):** Derived from historical data or formula
- **Manual (✏️):** User-entered rate (requires validation)
- **Market (📊):** Live market rate (most accurate)

### 4. Currency Formatting

Use `Intl.NumberFormat` for locale-aware currency formatting:

```typescript
const formatCurrency = (amount: number, currency: string) => {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: currency,
    minimumFractionDigits: 2,
    maximumFractionDigits: 2
  }).format(amount);
};
```

---

## Related Components

**Used by:**
- RelationshipPanel (embeds FXConversionCard when type = fx_conversion)
- FX Dashboard (displays multiple conversions in grid)
- Transaction Detail View (shows FX conversion for cross-border transactions)

**Uses:**
- None (primitive UI component)

**Related:**
- MetricCard (similar card-based display pattern)
- TransactionTable (shows transactions that trigger FX conversions)
- DrillDownPanel (links to detailed FX reports)

---

## File Paths

**Component location:**
`/Users/darwinborges/Description/src/components/primitives/il/FXConversionCard.tsx`

**Tests:**
`/Users/darwinborges/Description/tests/components/il/FXConversionCard.test.tsx`

**Storybook:**
`/Users/darwinborges/Description/stories/il/FXConversionCard.stories.tsx`

**Documentation:**
`/Users/darwinborges/Description/docs/primitives/il/FXConversionCard.md`
