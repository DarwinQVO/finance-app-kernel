# 4. Reports and Exports

**Goal:** Generate monthly summaries and export data

---

## What the user needs

**Monthly summary:**
- Total income for the month
- Total expenses for the month
- Net savings (income - expenses)
- Breakdown by category (top 5 spending categories)
- Month-over-month comparison

**Export transactions:**
- Download filtered transactions as CSV
- Use for tax preparation, accounting software, spreadsheets
- Include: Date, Description, Amount, Account, Category

**View past months:**
- See summary for any previous month
- Compare this month vs last month vs same month last year

---

## Scale expectations

- **Report generation:** Under 2 seconds
- **Export size:** ~500 rows per month (50KB CSV)
- **History:** Keep 2+ years of data
- **Frequency:** Generate monthly reports on-demand

---

## User experience

### Monthly summary dashboard

```
┌─────────────────────────────────────────────────┐
│ January 2025 Summary           [Export] [Print]│
├─────────────────────────────────────────────────┤
│                                                 │
│ Income:      $4,200  ↑ 5% vs last month        │
│ Expenses:    $3,100  ↓ 2% vs last month        │
│ Net Savings: $1,100  ↑ 12% vs last month       │
│                                                 │
├─────────────────────────────────────────────────┤
│ Top Spending Categories:                        │
│                                                 │
│ 🏠 Rent/Mortgage        $1,200  (39%)           │
│ 🍔 Food & Dining          $520  (17%)           │
│ 🚗 Transportation         $380  (12%)           │
│ ⚡ Utilities              $250  (8%)            │
│ 🎬 Entertainment          $180  (6%)            │
│                                                 │
└─────────────────────────────────────────────────┘

[< December]  [February >]
```

### Export dialog

```
┌─────────────────────────────────────────────────┐
│ Export Transactions                     [Close]│
├─────────────────────────────────────────────────┤
│ Date range:   [Jan 1, 2025] to [Jan 31, 2025] │
│ Accounts:     [All accounts ▼]                 │
│ Categories:   [All categories ▼]               │
│                                                 │
│ Format:       ● CSV                             │
│               ○ Excel (XLSX)                    │
│               ○ PDF                             │
│                                                 │
│ Include:      ☑ Date                            │
│               ☑ Description                     │
│               ☑ Amount                          │
│               ☑ Account                         │
│               ☑ Category                        │
│               ☐ Source file name                │
│                                                 │
│               [Export (42 transactions)]        │
└─────────────────────────────────────────────────┘
```

### CSV output example

```csv
Date,Description,Amount,Account,Category
2025-01-15,WHOLE FOODS MARKET,-87.43,Checking,Groceries
2025-01-14,PAYCHECK DEPOSIT,2100.00,Checking,Income
2025-01-12,PG&E ELECTRIC BILL,-124.00,Checking,Utilities
...
```

---

## Technical constraints (simple)

**Report calculation:**
- Simple SQL GROUP BY queries
- SUM(amount) WHERE date BETWEEN ... GROUP BY category
- No need for pre-aggregation or materialized views
- SQLite handles this easily for 500 tx/month

**CSV generation:**
- Python built-in csv module
- Stream directly to download (no temp files)
- Max file size: 500 rows = 50KB (negligible)

**Chart rendering:**
- Simple HTML/CSS bar charts (no JavaScript charting library needed initially)
- Or: Use basic Chart.js for pie chart
- No need for D3.js or complex visualizations

**Caching:**
- No caching needed (reports generate in <2 seconds)
- Recalculate on every page load (data is small)

---

## What's NOT needed (out of scope)

❌ Year-end tax summary (with tax categories)
❌ Forecasting / projections
❌ Budget tracking (planned vs actual spending)
❌ Spending trends over 12 months
❌ Custom report builder (drag & drop)
❌ Scheduled reports (email me monthly summary)
❌ Interactive charts (drill-down, zoom, pan)
❌ Multi-currency reports (USD only initially)
❌ Export to QuickBooks, Mint, YNAB formats
❌ Automatic categorization of exported data
