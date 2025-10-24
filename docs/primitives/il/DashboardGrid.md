# DashboardGrid (IL Component)

## Definition

**DashboardGrid** is a responsive grid layout system for dashboard widgets. It provides a 12-column grid on desktop and stacks widgets vertically on mobile, with drag-and-drop support for custom layouts (v2 feature).

**Problem it solves:**
- Dashboard widgets need consistent spacing and alignment
- Mobile layouts require different arrangement than desktop
- Manual CSS grid configuration is error-prone
- No standard way to handle widget positioning across devices

**Solution:**
- 12-column grid system (like Bootstrap/Tailwind)
- Responsive breakpoints (desktop, tablet, mobile)
- Widget positioning via row/col/width configuration
- Automatic stacking on mobile (no horizontal scroll)
- Collision detection (prevent overlapping widgets)

---

## Interface Contract

```typescript
interface DashboardGridProps {
  // Widgets configuration
  widgets: WidgetConfig[];

  // Callbacks
  onWidgetClick?: (widgetType: string) => void;  // Drill-down from metric
  onLayoutChange?: (newWidgets: WidgetConfig[]) => void;  // Drag-and-drop (v2)

  // UI customization
  gap?: number;  // Spacing between widgets (default: 16px)
  columns?: number;  // Desktop columns (default: 12)
  responsive?: boolean;  // Enable responsive layout (default: true)
  theme?: "light" | "dark";
}

interface WidgetConfig {
  type: string;  // "income_summary", "expense_by_category", etc.
  position: {
    row: number;     // Row index (0-indexed)
    col: number;     // Column index (0-11 for 12-col grid)
    width: number;   // Span across N columns (1-12)
  };
  data?: any;  // Widget-specific data (passed from DashboardEngine)
}

interface DashboardGridLayout {
  desktop: { columns: 12, gap: 16 };
  tablet: { columns: 6, gap: 12 };
  mobile: { columns: 1, gap: 8 };  // Stack vertically
}
```

---

## Component Structure

```tsx
import React from 'react';

export const DashboardGrid: React.FC<DashboardGridProps> = ({
  widgets,
  onWidgetClick,
  onLayoutChange,
  gap = 16,
  columns = 12,
  responsive = true,
  theme = "light"
}) => {
  // Responsive layout calculation
  const getGridLayout = (): DashboardGridLayout => {
    const width = window.innerWidth;
    if (width < 768) return { columns: 1, gap: 8 };  // Mobile
    if (width < 1024) return { columns: 6, gap: 12 };  // Tablet
    return { columns: 12, gap: 16 };  // Desktop
  };

  const [layout, setLayout] = useState(getGridLayout());

  useEffect(() => {
    if (!responsive) return;

    const handleResize = () => setLayout(getGridLayout());
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, [responsive]);

  // Render grid
  return (
    <div
      className={`dashboard-grid theme-${theme}`}
      style={{
        display: 'grid',
        gridTemplateColumns: `repeat(${layout.columns}, 1fr)`,
        gap: `${layout.gap}px`,
        padding: `${gap}px`
      }}
    >
      {widgets.map((widget, index) => (
        <div
          key={index}
          className="dashboard-widget"
          style={{
            gridColumn: layout.columns === 1
              ? 'span 1'  // Mobile: full width
              : `${widget.position.col + 1} / span ${widget.position.width}`,
            gridRow: widget.position.row + 1
          }}
          onClick={() => onWidgetClick?.(widget.type)}
        >
          {renderWidget(widget)}
        </div>
      ))}
    </div>
  );
};

// Render individual widget (delegates to MetricCard, CategoryBreakdown, etc.)
function renderWidget(widget: WidgetConfig): React.ReactNode {
  switch (widget.type) {
    case 'income_summary':
      return <MetricCard type="income" data={widget.data} />;
    case 'expense_summary':
      return <MetricCard type="expense" data={widget.data} />;
    case 'category_breakdown':
      return <CategoryBreakdownTable data={widget.data} />;
    case 'top_merchants':
      return <TopMerchantsCard data={widget.data} />;
    default:
      return <div>Unknown widget type: {widget.type}</div>;
  }
}
```

---

## Multi-Domain Applicability

### Finance Domain
```tsx
<DashboardGrid
  widgets={[
    {type: "income_summary", position: {row: 0, col: 0, width: 4}, data: {...}},
    {type: "expense_summary", position: {row: 0, col: 4, width: 4}, data: {...}},
    {type: "net_summary", position: {row: 0, col: 8, width: 4}, data: {...}},
    {type: "category_breakdown", position: {row: 1, col: 0, width: 8}, data: {...}},
    {type: "top_merchants", position: {row: 1, col: 8, width: 4}, data: {...}}
  ]}
  onWidgetClick={(type) => console.log(`Clicked ${type}`)}
/>
```

### Healthcare Domain
```tsx
<DashboardGrid
  widgets={[
    {type: "avg_glucose", position: {row: 0, col: 0, width: 4}, data: {...}},
    {type: "avg_blood_pressure", position: {row: 0, col: 4, width: 4}, data: {...}},
    {type: "abnormal_tests", position: {row: 0, col: 8, width: 4}, data: {...}},
    {type: "vitals_trend", position: {row: 1, col: 0, width: 12}, data: {...}}
  ]}
/>
```

### Legal Domain
```tsx
<DashboardGrid
  widgets={[
    {type: "cases_won", position: {row: 0, col: 0, width: 3}, data: {...}},
    {type: "cases_lost", position: {row: 0, col: 3, width: 3}, data: {...}},
    {type: "cases_settled", position: {row: 0, col: 6, width: 3}, data: {...}},
    {type: "win_rate", position: {row: 0, col: 9, width: 3}, data: {...}},
    {type: "hours_by_client", position: {row: 1, col: 0, width: 12}, data: {...}}
  ]}
/>
```

---

## Responsive Behavior

**Desktop (≥1024px):** 12-column grid
```
┌────────────┬────────────┬────────────┐
│  Income    │  Expenses  │     Net    │
│  (col 0-3) │ (col 4-7)  │  (col 8-11)│
├──────────────────────┬──────────────┤
│  Category Breakdown  │ Top Merchants│
│     (col 0-7)        │  (col 8-11)  │
└──────────────────────┴──────────────┘
```

**Tablet (768-1023px):** 6-column grid
```
┌──────────┬──────────┬──────────┐
│  Income  │ Expenses │    Net   │
│ (col 0-1)│ (col 2-3)│ (col 4-5)│
├──────────────────────────────┤
│    Category Breakdown        │
│         (col 0-5)            │
├──────────────────────────────┤
│      Top Merchants           │
│         (col 0-5)            │
└──────────────────────────────┘
```

**Mobile (<768px):** 1-column (stacked)
```
┌──────────────────────┐
│       Income         │
├──────────────────────┤
│      Expenses        │
├──────────────────────┤
│         Net          │
├──────────────────────┤
│ Category Breakdown   │
├──────────────────────┤
│   Top Merchants      │
└──────────────────────┘
```

---

## Styling

```css
.dashboard-grid {
  width: 100%;
  max-width: 1400px;
  margin: 0 auto;
}

.dashboard-widget {
  background: var(--widget-bg);
  border: 1px solid var(--widget-border);
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: transform 0.2s, box-shadow 0.2s;
  cursor: pointer;
}

.dashboard-widget:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

/* Theme: Light */
.theme-light {
  --widget-bg: #ffffff;
  --widget-border: #e5e7eb;
}

/* Theme: Dark */
.theme-dark {
  --widget-bg: #1f2937;
  --widget-border: #374151;
}

/* Responsive breakpoints */
@media (max-width: 768px) {
  .dashboard-grid {
    padding: 8px;
  }

  .dashboard-widget {
    padding: 12px;
  }
}
```

---

## Collision Detection

```typescript
function detectCollisions(widgets: WidgetConfig[]): boolean {
  for (let i = 0; i < widgets.length; i++) {
    for (let j = i + 1; j < widgets.length; j++) {
      const w1 = widgets[i].position;
      const w2 = widgets[j].position;

      // Check if widgets are on same row
      if (w1.row === w2.row) {
        // Check if columns overlap
        const w1_end = w1.col + w1.width;
        const w2_end = w2.col + w2.width;

        if (
          (w1.col >= w2.col && w1.col < w2_end) ||
          (w2.col >= w1.col && w2.col < w1_end)
        ) {
          console.error(`Collision detected between widgets at row ${w1.row}`);
          return true;
        }
      }
    }
  }
  return false;
}
```

---

## Reusable Across Domains

**DashboardGrid is domain-agnostic.** It only cares about widget positioning and responsive layout. The actual widget content is determined by the widget type and data passed in.

**Same grid, different widgets:**
- Finance: MetricCard (income, expenses, net)
- Healthcare: VitalsCard (glucose, blood pressure, heart rate)
- Legal: CaseMetricCard (won, lost, settled)
- Research: PublicationMetricCard (papers, citations, h-index)
- Manufacturing: ProductionMetricCard (units, defects, pass rate)
- Media: EngagementMetricCard (views, likes, watch time)

---

## Related Components

**Uses:**
- MetricCard (renders individual metric widgets)
- CategoryBreakdownTable (renders category breakdown widget)
- TopMerchantsCard (renders top merchants widget)

**Used by:**
- Dashboard page (/dashboard)
- Custom dashboard editor (v2 feature)
