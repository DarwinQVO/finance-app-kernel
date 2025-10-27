# MetricCard (IL Component)

## Definition

**MetricCard** is a reusable widget for displaying a single metric (number, trend, breakdown). It provides consistent styling and interaction patterns across all dashboard widgets.

**Problem it solves:**
- Inconsistent metric display across different dashboards
- No standard format for showing numbers + trends + breakdowns
- Tedious to implement loading/error/empty states for each metric
- No click-to-drill-down pattern

**Solution:**
- Single component for all metric types
- Consistent styling (card layout, typography, colors)
- Built-in states (loading, error, empty)
- Click-to-drill-down support
- Responsive design (compact on mobile)

---

## Interface Contract

```typescript
interface MetricCardProps {
  // Metric data
  type: "income" | "expense" | "net" | "custom";
  label: string;  // "Total Income", "Expenses", etc.
  value: number | string;  // Main metric value
  currency?: string;  // "USD", "MXN" (default: "USD")

  // Optional breakdown
  breakdown?: Array<{
    label: string;
    value: number;
    percentage?: number;
  }>;

  // Optional trend
  trend?: {
    direction: "up" | "down" | "flat";
    percentage: number;
    comparison: string;  // "vs last month", "vs Q1 2024"
  };

  // UI states
  loading?: boolean;
  error?: string | null;
  empty?: boolean;

  // Interaction
  onClick?: () => void;  // Drill-down handler
  clickable?: boolean;  // Show hover effect + cursor pointer

  // Customization
  theme?: "light" | "dark";
  size?: "small" | "medium" | "large";
  showBreakdown?: boolean;  // Show/hide breakdown
}
```

---

## Component Structure

```tsx
import React from 'react';

export const MetricCard: React.FC<MetricCardProps> = ({
  type,
  label,
  value,
  currency = "USD",
  breakdown,
  trend,
  loading = false,
  error = null,
  empty = false,
  onClick,
  clickable = true,
  theme = "light",
  size = "medium",
  showBreakdown = true
}) => {
  // Loading state
  if (loading) {
    return (
      <div className={`metric-card metric-card-${size} theme-${theme} loading`}>
        <div className="skeleton skeleton-label"></div>
        <div className="skeleton skeleton-value"></div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className={`metric-card metric-card-${size} theme-${theme} error`}>
        <div className="error-icon">⚠️</div>
        <div className="error-message">{error}</div>
      </div>
    );
  }

  // Empty state
  if (empty) {
    return (
      <div className={`metric-card metric-card-${size} theme-${theme} empty`}>
        <div className="empty-icon">📊</div>
        <div className="empty-message">No data for this period</div>
      </div>
    );
  }

  // Format value
  const formattedValue = typeof value === 'number'
    ? new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency: currency,
        minimumFractionDigits: 0,
        maximumFractionDigits: 0
      }).format(value)
    : value;

  return (
    <div
      className={`metric-card metric-card-${size} metric-card-${type} theme-${theme} ${clickable ? 'clickable' : ''}`}
      onClick={clickable ? onClick : undefined}
    >
      {/* Label */}
      <div className="metric-label">{label}</div>

      {/* Main value */}
      <div className="metric-value">{formattedValue}</div>

      {/* Trend indicator (optional) */}
      {trend && (
        <div className={`metric-trend trend-${trend.direction}`}>
          <span className="trend-icon">
            {trend.direction === 'up' && '↑'}
            {trend.direction === 'down' && '↓'}
            {trend.direction === 'flat' && '→'}
          </span>
          <span className="trend-percentage">{trend.percentage}%</span>
          <span className="trend-comparison">{trend.comparison}</span>
        </div>
      )}

      {/* Breakdown (optional) */}
      {showBreakdown && breakdown && breakdown.length > 0 && (
        <div className="metric-breakdown">
          {breakdown.slice(0, 3).map((item, index) => (
            <div key={index} className="breakdown-item">
              <span className="breakdown-label">{item.label}</span>
              <span className="breakdown-value">
                {new Intl.NumberFormat('en-US', {
                  style: 'currency',
                  currency: currency,
                  minimumFractionDigits: 0
                }).format(item.value)}
              </span>
              {item.percentage && (
                <span className="breakdown-percentage">({item.percentage}%)</span>
              )}
            </div>
          ))}
          {breakdown.length > 3 && (
            <div className="breakdown-more">+{breakdown.length - 3} more</div>
          )}
        </div>
      )}
    </div>
  );
};
```

---

## Styling

```css
.metric-card {
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: 8px;
  padding: 16px;
  transition: transform 0.2s, box-shadow 0.2s;
}

.metric-card.clickable {
  cursor: pointer;
}

.metric-card.clickable:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

.metric-label {
  font-size: 14px;
  font-weight: 500;
  color: var(--label-color);
  margin-bottom: 8px;
}

.metric-value {
  font-size: 32px;
  font-weight: 700;
  color: var(--value-color);
  margin-bottom: 8px;
}

/* Size variants */
.metric-card-small .metric-value { font-size: 24px; }
.metric-card-medium .metric-value { font-size: 32px; }
.metric-card-large .metric-value { font-size: 48px; }

/* Type-specific colors */
.metric-card-income .metric-value { color: #10b981; }  /* Green */
.metric-card-expense .metric-value { color: #ef4444; }  /* Red */
.metric-card-net .metric-value { color: #3b82f6; }  /* Blue */

/* Trend indicator */
.metric-trend {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  margin-bottom: 8px;
}

.trend-up { color: #10b981; }
.trend-down { color: #ef4444; }
.trend-flat { color: #6b7280; }

/* Breakdown */
.metric-breakdown {
  border-top: 1px solid var(--card-border);
  padding-top: 12px;
  margin-top: 12px;
}

.breakdown-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
  font-size: 13px;
}

.breakdown-label {
  color: var(--label-color);
}

.breakdown-value {
  font-weight: 600;
  color: var(--value-color);
}

.breakdown-percentage {
  color: var(--secondary-color);
  margin-left: 4px;
}

/* Loading skeleton */
.skeleton {
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: loading 1.5s infinite;
  border-radius: 4px;
}

.skeleton-label {
  height: 20px;
  width: 60%;
  margin-bottom: 8px;
}

.skeleton-value {
  height: 40px;
  width: 80%;
}

@keyframes loading {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

/* Theme: Light */
.theme-light {
  --card-bg: #ffffff;
  --card-border: #e5e7eb;
  --label-color: #6b7280;
  --value-color: #111827;
  --secondary-color: #9ca3af;
}

/* Theme: Dark */
.theme-dark {
  --card-bg: #1f2937;
  --card-border: #374151;
  --label-color: #9ca3af;
  --value-color: #f3f4f6;
  --secondary-color: #6b7280;
}
```

---

## Multi-Domain Applicability

### Finance Domain
```tsx
<MetricCard
  type="income"
  label="Total Income"
  value={9000.00}
  currency="USD"
  trend={{direction: "up", percentage: 12, comparison: "vs last month"}}
  breakdown={[
    {label: "HubSpot Inc", value: 5000, percentage: 56},
    {label: "Stripe Transfer", value: 4000, percentage: 44}
  ]}
  onClick={() => drillDownToTransactions("income")}
/>
```

### Healthcare Domain
```tsx
<MetricCard
  type="custom"
  label="Average Glucose"
  value="120 mg/dL"
  trend={{direction: "down", percentage: 5, comparison: "vs last week"}}
  breakdown={[
    {label: "Fasting", value: 110},
    {label: "Post-meal", value: 130}
  ]}
  onClick={() => viewLabResults()}
/>
```

### Legal Domain
```tsx
<MetricCard
  type="custom"
  label="Cases Won"
  value={45}
  trend={{direction: "up", percentage: 8, comparison: "vs Q4 2024"}}
  breakdown={[
    {label: "Civil", value: 30, percentage: 67},
    {label: "Criminal", value: 15, percentage: 33}
  ]}
  onClick={() => viewCaseList("won")}
/>
```

### Research Domain
```tsx
<MetricCard
  type="custom"
  label="Papers Published"
  value={12}
  trend={{direction: "up", percentage: 20, comparison: "vs 2024"}}
  breakdown={[
    {label: "Peer-reviewed", value: 9, percentage: 75},
    {label: "Preprints", value: 3, percentage: 25}
  ]}
  onClick={() => viewPublications()}
/>
```

### Manufacturing Domain
```tsx
<MetricCard
  type="custom"
  label="Defect Rate"
  value="0.87%"
  trend={{direction: "down", percentage: 15, comparison: "vs last month"}}
  breakdown={[
    {label: "Assembly defects", value: 45},
    {label: "Material defects", value: 32}
  ]}
  onClick={() => viewQCReport()}
/>
```

### Media Domain
```tsx
<MetricCard
  type="custom"
  label="Total Views"
  value={450000}
  trend={{direction: "up", percentage: 25, comparison: "vs last month"}}
  breakdown={[
    {label: "YouTube", value: 320000, percentage: 71},
    {label: "TikTok", value: 130000, percentage: 29}
  ]}
  onClick={() => viewAnalytics()}
/>
```

---

## Variants

### 1. Simple MetricCard (no breakdown, no trend)
```tsx
<MetricCard
  type="net"
  label="Net Profit"
  value={753.00}
  currency="USD"
/>
```

### 2. Trend-only MetricCard
```tsx
<MetricCard
  type="expense"
  label="Monthly Expenses"
  value={8247.00}
  currency="USD"
  trend={{direction: "down", percentage: 3, comparison: "vs last month"}}
/>
```

### 3. Breakdown-only MetricCard
```tsx
<MetricCard
  type="custom"
  label="Deductible Expenses"
  value={3450.00}
  currency="USD"
  breakdown={[
    {label: "Software", value: 2500, percentage: 72},
    {label: "Travel", value: 950, percentage: 28}
  ]}
/>
```

---

## Accessibility

```tsx
<div
  className="metric-card"
  role="button"
  tabIndex={clickable ? 0 : undefined}
  aria-label={`${label}: ${formattedValue}`}
  onKeyPress={(e) => {
    if (e.key === 'Enter' && clickable) onClick?.();
  }}
>
  {/* Card content */}
</div>
```

---

## Related Components

**Used by:**
- DashboardGrid (renders MetricCard widgets)
- Custom dashboard builder (v2)

**Uses:**
- None (primitive UI component)
