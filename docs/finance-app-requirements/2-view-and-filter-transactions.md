# 2. View and Filter Transactions

**Goal:** See all transactions and filter to find specific ones

---

## What the user needs

**View all transactions:**
- Table showing: Date | Description | Amount | Account | Category
- Most recent transactions first
- Show 50-100 transactions per page
- Pagination for older transactions

**Filter transactions:**
- By date range (last month, last 3 months, custom range)
- By account (checking, savings, credit card)
- By category (groceries, rent, utilities, etc.)
- By amount range (transactions over $100)
- Free text search in description
- "Reset filters" button to clear all

**Summary totals:**
- Total income (filtered)
- Total expenses (filtered)
- Net balance (filtered)
- Show at top of page

**Click transaction to see details:**
- Full description
- Which PDF it came from
- When it was imported
- Any corrections made

---

## Scale expectations

- **Transactions displayed:** 50-100 per page
- **Total transactions:** 6,000 per year
- **Filter response time:** Under 1 second
- **Concurrent users:** 1 (single user app)

---

## User experience

### Main transaction list

```
┌─────────────────────────────────────────────────┐
│ Transactions                    [Upload] [Export]│
├─────────────────────────────────────────────────┤
│ Filters:                                        │
│ [Last 30 days ▼] [All Accounts ▼] [Search...] │
│ [Reset Filters]                                 │
├─────────────────────────────────────────────────┤
│ Income: $4,200  Expenses: $3,100  Net: $1,100  │
├─────────────────────────────────────────────────┤
│ Date       Description      Amount    Account   │
├─────────────────────────────────────────────────┤
│ Jan 15     Grocery Store   -$87.43   Checking   │
│ Jan 14     Paycheck       +$2,100    Checking   │
│ Jan 12     Electric Bill   -$124.00  Checking   │
│ ...                                              │
├─────────────────────────────────────────────────┤
│                        [Prev] Page 1 of 4 [Next]│
└─────────────────────────────────────────────────┘
```

### Detail view (click transaction)

```
┌─────────────────────────────────────────────────┐
│ Transaction Details                      [Close]│
├─────────────────────────────────────────────────┤
│ Date:         January 15, 2025                  │
│ Description:  WHOLE FOODS #1234 SAN FRANCISCO   │
│ Amount:       -$87.43                           │
│ Account:      Checking (****1234)               │
│ Category:     Groceries                         │
│                                                 │
│ Source:       statement-jan-2025.pdf (page 2)  │
│ Imported:     Jan 16, 2025 at 3:24 PM          │
│                                                 │
│             [View Source PDF] [Edit Category]   │
└─────────────────────────────────────────────────┘
```

---

## Technical constraints (simple)

**Pagination:**
- Simple offset-based pagination (not cursor-based)
- 500 tx/month = ~10 pages max with 50 tx/page
- No need for complex virtualization

**Filters:**
- Simple SQL WHERE clauses
- No need for ElasticSearch or full-text search engines
- Basic LIKE queries for text search are sufficient

**Performance:**
- SQLite can easily handle 6,000 transactions
- Indexes on: date, account_id, category_id
- No caching layer needed (Redis not required)

**Sorting:**
- Default: date DESC (newest first)
- Allow sorting by: date, amount, description
- Simple SQL ORDER BY is sufficient

---

## What's NOT needed (out of scope)

❌ Real-time updates (refresh page to see new transactions)
❌ Advanced search syntax (boolean operators, regex)
❌ Save custom filter views initially
❌ Share filtered views with others
❌ Bulk actions (select multiple, delete multiple)
❌ Infinite scroll (pagination is fine)
❌ Excel-like pivot tables
❌ Advanced analytics dashboards
