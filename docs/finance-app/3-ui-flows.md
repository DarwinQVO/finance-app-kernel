# UI Flows

> **Purpose:** Define user interface screens and interactions (wireframes + user actions)

---

## Screen 1: File Upload

**One Button Philosophy:** User shouldn't need a manual

**Desktop Wireframe:**

```
┌────────────────────────────────────────────────────────────────┐
│ Finance App                      [Dashboard] [Transactions] [▼]│
├────────────────────────────────────────────────────────────────┤
│                                                                │
│              📄                                                │
│         Upload Statement                                       │
│                                                                │
│   ┌──────────────────────────────────────────────────────┐   │
│   │                                                       │   │
│   │         Drag & Drop PDF here                         │   │
│   │              or                                       │   │
│   │         [Browse Files...]                            │   │
│   │                                                       │   │
│   │  Supported: Bank of America statements (PDF only)    │   │
│   │                                                       │   │
│   └──────────────────────────────────────────────────────┘   │
│                                                                │
│   Recent Uploads:                                              │
│   ┌──────────────────────────────────────────────────────┐   │
│   │ ✓ bofa-october.pdf      Oct 27, 10:30 AM  42 txns   │   │
│   │ ✓ bofa-september.pdf    Oct  1, 09:15 AM  38 txns   │   │
│   │ ⚠ bofa-august.pdf       Sep  5, 11:20 AM  Duplicate │   │
│   └──────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**Mobile Wireframe:**

```
┌──────────────────────┐
│ Upload        ☰     │
├──────────────────────┤
│                      │
│      📄              │
│  Upload Statement    │
│                      │
│ ┌──────────────────┐ │
│ │  Drag & Drop     │ │
│ │       or         │ │
│ │ [Choose File...] │ │
│ └──────────────────┘ │
│                      │
│ Recent:              │
│ ┌──────────────────┐ │
│ │✓ bofa-oct.pdf    │ │
│ │  42 transactions │ │
│ ├──────────────────┤ │
│ │✓ bofa-sep.pdf    │ │
│ │  38 transactions │ │
│ └──────────────────┘ │
│                      │
└──────────────────────┘
```

**User Actions:**

1. **Drop file or click Browse**
   - File picker opens (desktop: native, mobile: camera/files)
   - User selects `bofa-statement-october.pdf`

2. **Upload begins**
   ```
   ┌──────────────────────────────────────────┐
   │ Uploading bofa-statement-october.pdf     │
   │ ████████████░░░░░░░░░░░░░░░ 45%  1.2 MB │
   └──────────────────────────────────────────┘
   ```

3. **Processing status**
   ```
   ┌──────────────────────────────────────────┐
   │ ✓ Upload complete                        │
   │ ⏳ Processing statement... ~30 seconds   │
   │                                          │
   │ [View Dashboard]  [Upload Another]      │
   └──────────────────────────────────────────┘
   ```

4. **Error states**
   ```
   ❌ Upload Failed: File is not a PDF
   ❌ Processing Error: Could not read PDF - file may be corrupted
   ⚠️  Duplicate: This file was uploaded on Oct 15
   ```

---

## Screen 2: Transaction List (Main Screen)

**Table with Filters:** Classic financial app layout

**Desktop Wireframe:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Transactions                    [Upload] [Export] [Settings]            │
├──────────────────────────────────────────────────────────────────────────┤
│ Filters:                                                                 │
│ [Last 30 days ▼] [All Accounts ▼] [All Categories ▼] [Search...       ]│
│ [Reset Filters]                                                          │
├──────────────────────────────────────────────────────────────────────────┤
│ Income: $4,200  │  Expenses: $3,100  │  Net: $1,100                    │
├──────────────────────────────────────────────────────────────────────────┤
│ Date       Description             Amount    Account      Category      │
├──────────────────────────────────────────────────────────────────────────┤
│ Oct 27     WHOLE FOODS MARKET    -$87.43    Checking     Groceries      │
│ Oct 26     SALARY DEPOSIT       +$2,100     Checking     Income         │
│ Oct 25     NETFLIX.COM           -$14.99    Credit Card  Entertainment  │
│ Oct 24     SHELL GAS #1234       -$45.00    Checking     Transportation │
│ Oct 23     STARBUCKS #456         -$5.25    Credit Card  Food & Dining  │
│ Oct 22     PG&E ELECTRIC        -$124.00    Checking     Utilities      │
│ Oct 21     SAFEWAY STORE          -$62.18   Checking     Groceries      │
│ Oct 20     UBER TRIP              -$18.50   Credit Card  Transportation │
│ Oct 19     AMAZON MKTPLACE        -$45.99   Credit Card  Shopping       │
│ Oct 18     VERIZON WIRELESS       -$85.00   Checking     Utilities      │
│                                                                          │
│ Showing 1-50 of 127 transactions              [< Prev]  [1] 2 3  [Next >]│
└──────────────────────────────────────────────────────────────────────────┘
```

**Mobile Wireframe:**

```
┌────────────────────────┐
│ Transactions    ☰     │
├────────────────────────┤
│ [Filters...] [+Upload]│
│                        │
│ Income:  $4,200        │
│ Expense: $3,100        │
│ Net:     $1,100        │
├────────────────────────┤
│ Oct 27                 │
│ WHOLE FOODS            │
│ -$87.43    Groceries   │
├────────────────────────┤
│ Oct 26                 │
│ SALARY DEPOSIT         │
│ +$2,100    Income      │
├────────────────────────┤
│ Oct 25                 │
│ NETFLIX.COM            │
│ -$14.99    Streaming   │
├────────────────────────┤
│ [Load More...]         │
│                        │
└────────────────────────┘
```

**User Actions:**

1. **Filter by date**
   - Click "[Last 30 days ▼]" → Dropdown opens
   - Options: This Month, Last Month, Last 3 Months, Custom Range, All Time
   - Select "This Month" → Table refreshes

2. **Filter by account**
   - Click "[All Accounts ▼]" → Dropdown opens
   - Options: All Accounts, Chase Checking, Chase Savings, BoFA Credit Card, etc.
   - Select "Chase Checking" → Table shows only checking transactions

3. **Search**
   - Type "starbucks" in search box
   - Table filters to show only Starbucks transactions
   - Clear search → Shows all transactions again

4. **Click transaction**
   - Modal opens showing transaction details (see Screen 3)

5. **Sort columns**
   - Click "Date" column header → Sort by date descending
   - Click "Amount" column header → Sort by amount descending
   - Arrow indicator shows sort direction ▼ ▲

6. **Pagination**
   - Click "Next >" → Shows transactions 51-100
   - Click page number "2" → Jumps to page 2
   - Simple offset pagination (not cursor-based for simplicity)

---

## Screen 3: Transaction Detail Modal

**Full Transparency:** Show everything about this transaction

**Desktop Wireframe:**

```
┌──────────────────────────────────────────────────────────────────┐
│ Transaction Details                                    [✕ Close] │
├──────────────────────────────────────────────────────────────────┤
│ ┌─ Canonical Data ─┬─ Raw Data ─┬─ Decisions ─┬─ Source PDF ─┐ │
│ │                                                               │ │
│ │  Date:         October 15, 2024                              │ │
│ │  Merchant:     Whole Foods Market                            │ │
│ │  Amount:       -$87.43                                       │ │
│ │  Category:     Groceries                     [Edit]          │ │
│ │  Account:      Chase Checking (****1234)                     │ │
│ │  Confidence:   High (0.95)                                   │ │
│ │                                                               │ │
│ │  Notes:        [Add notes...]                                │ │
│ │  Tags:         groceries, weekly              [+ Add tag]    │ │
│ │                                                               │ │
│ │  ┌─────────────────────────────────────────────────────┐    │ │
│ │  │ Created:   Oct 16, 2024 at 10:30 AM                │    │ │
│ │  │ Modified:  Oct 20, 2024 at 3:15 PM (category edit) │    │ │
│ │  │ Source:    bofa-statement-october.pdf (page 2)     │    │ │
│ │  └─────────────────────────────────────────────────────┘    │ │
│ │                                                               │ │
│ │  [View Raw Data] [View Decisions] [View Source PDF]         │ │
│ │                                                               │ │
│ └───────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  [Delete Transaction] [Duplicate] [Flag for Review]  [Close]    │
└──────────────────────────────────────────────────────────────────┘
```

**Tab 2: Raw Data (what PDF said):**

```
┌──────────────────────────────────────────┐
│ Raw Observation from PDF:                │
│                                          │
│ Date:         "10/15/2024"               │
│ Description:  "WHOLE FOODS MARKET #1234  │
│                SAN FRANCISCO CA"         │
│ Amount:       "$87.43"                   │
│ Account:      "****1234"                 │
│                                          │
│ This is exactly what appeared in the PDF │
│ before any processing or cleaning.       │
└──────────────────────────────────────────┘
```

**Tab 3: Normalization Decisions:**

```
┌─────────────────────────────────────────────────────┐
│ How we processed the raw data:                      │
│                                                     │
│ 1. Date Cleaning:                                   │
│    Raw: "10/15/2024"                                │
│    → Parsed as MM/DD/YYYY format                    │
│    → Result: 2024-10-15 (ISO 8601)                  │
│                                                     │
│ 2. Merchant Normalization:                          │
│    Raw: "WHOLE FOODS MARKET #1234 SAN FRANCISCO CA" │
│    → Stripped store number and location             │
│    → Result: "Whole Foods Market"                   │
│    Rule: merchant_normalization_rules.yaml line 42  │
│                                                     │
│ 3. Amount Parsing:                                  │
│    Raw: "$87.43"                                    │
│    → Removed "$" symbol                             │
│    → Negated (expense transaction)                  │
│    → Result: -87.43                                 │
│                                                     │
│ 4. Category Assignment:                             │
│    Merchant: "Whole Foods Market"                   │
│    → Pattern matched: "WHOLE FOODS"                 │
│    → Result: Category "Groceries"                   │
│    Rule: categorization_rules.yaml line 15          │
│    Confidence: 0.90                                 │
│                                                     │
│ 5. Account Resolution:                              │
│    Raw: "****1234"                                  │
│    → Looked up in Account Registry                  │
│    → Result: Chase Checking (ACC_chase_checking)    │
└─────────────────────────────────────────────────────┘
```

**Tab 4: Source PDF:**

```
┌──────────────────────────────────────────┐
│ bofa-statement-october.pdf - Page 2      │
│                                          │
│ [PDF Viewer with highlighted row]        │
│                                          │
│ Date       Description        Amount     │
│ 10/14/2024 NETFLIX.COM        $14.99    │
│ 10/15/2024 WHOLE FOODS...     $87.43  ← │
│ 10/16/2024 SHELL GAS #123     $45.00    │
│                                          │
│ [Download PDF] [Print]                   │
└──────────────────────────────────────────┘
```

**User Actions:**

1. **Edit category**
   - Click [Edit] next to Category
   - Dropdown opens with category list
   - Select new category → Saves immediately
   - Toast notification: "Category updated to Books & Education"
   - Audit log entry created

2. **Add note**
   - Click in Notes field
   - Type "Split with roommate - owe me $43.71"
   - Click outside field → Auto-saves
   - Note appears immediately

3. **Add tag**
   - Click "[+ Add tag]"
   - Type "reimbursable"
   - Press Enter → Tag added
   - Can add multiple tags

4. **Delete transaction**
   - Click [Delete Transaction]
   - Confirmation dialog: "Are you sure? This cannot be undone."
   - User confirms → Transaction soft-deleted (active=false)
   - Removed from list view

---

## Screen 4: Category Manager

**Simple CRUD:** Create, read, update, delete categories

**Desktop Wireframe:**

```
┌──────────────────────────────────────────────────────────────┐
│ Categories                              [+ New Category]     │
├──────────────────────────────────────────────────────────────┤
│ Search: [                               ]  [Show Archived]  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ 📂 Groceries (42 transactions this month)     ● #4CAF50     │
│    └─ Supermarkets (28 txns)                                │
│    └─ Farmers Markets (8 txns)                              │
│    └─ Specialty Stores (6 txns)                             │
│                                              [Edit] [Delete] │
│                                                              │
│ 🏠 Rent/Mortgage (1 transaction)             ● #2196F3     │
│                                              [Edit] [Delete] │
│                                                              │
│ 🚗 Transportation (18 transactions)          ● #FF9800     │
│    └─ Gas (12 txns)                                         │
│    └─ Parking (4 txns)                                      │
│    └─ Uber/Lyft (2 txns)                                    │
│                                              [Edit] [Delete] │
│                                                              │
│ 🍔 Food & Dining (35 transactions)           ● #F44336     │
│    └─ Restaurants (22 txns)                                 │
│    └─ Coffee Shops (10 txns)                                │
│    └─ Fast Food (3 txns)                                    │
│                                              [Edit] [Delete] │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Create New Category Dialog:**

```
┌──────────────────────────────────────────┐
│ New Category                   [✕ Close] │
├──────────────────────────────────────────┤
│                                          │
│ Name: [Coffee Shops              ]       │
│                                          │
│ Parent Category:                         │
│ [Food & Dining ▼]                        │
│                                          │
│ Color:                                   │
│ ● Red     ● Blue    ● Green             │
│ ● Orange  ● Purple  ● Yellow            │
│ Custom: [#FF5722]  [Color Picker]       │
│                                          │
│ Icon (optional): ☕                      │
│                                          │
│         [Cancel]  [Create Category]      │
└──────────────────────────────────────────┘
```

**User Actions:**

1. **Create subcategory**
   - Click [+ New Category]
   - Name: "Coffee Shops"
   - Parent: "Food & Dining"
   - Color: Orange
   - Click [Create] → Category added to tree

2. **Edit category**
   - Click [Edit] next to "Groceries"
   - Change name, color, or parent
   - Click [Save] → Updates immediately

3. **Delete category**
   - Click [Delete] next to "Coffee Shops"
   - Warning: "This category is used by 10 transactions. What should we do?"
   - Options:
     - Move transactions to parent category (Food & Dining)
     - Assign to different category
     - Leave uncategorized
   - User selects option → Category deleted, transactions updated

4. **Archive category**
   - For categories no longer used but with historical data
   - Click [Archive] → Hidden from dropdowns but preserved
   - Toggle [Show Archived] to view

---

## Screen 5: Dashboard (3 Charts)

**At-a-Glance Summary:** Monthly spending overview

**Desktop Wireframe:**

```
┌────────────────────────────────────────────────────────────────────┐
│ Dashboard                              [This Month ▼] [Export PDF] │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│ ┌─────────────────────────────────────────────────────────────┐  │
│ │ October 2024 Summary                                        │  │
│ │                                                             │  │
│ │  Income:      $4,200  ↑ 5% vs Sept                         │  │
│ │  Expenses:    $3,100  ↓ 2% vs Sept                         │  │
│ │  Net Savings: $1,100  ↑ 12% vs Sept                        │  │
│ │                                                             │  │
│ └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│ ┌─────────────────────────────┐ ┌──────────────────────────────┐ │
│ │ Top Spending Categories     │ │ Spending Trend (6 months)    │ │
│ │                             │ │                              │ │
│ │ Rent      $1,200  █████████ │ │   $3,500 ┤     ●─●          │ │
│ │ Groceries   $520  ███       │ │   $3,000 ┤   ●   ●          │ │
│ │ Transport   $380  ██        │ │   $2,500 ┤ ●       ●─●      │ │
│ │ Utilities   $250  █         │ │   $2,000 ┤                  │ │
│ │ Entertain   $180  █         │ │   $1,500 ┤                  │ │
│ │ Other       $570  ██        │ │          └─────────────────  │ │
│ │                             │ │          May Jun Jul Aug Sep │ │
│ │ [View All Categories]       │ │                              │ │
│ └─────────────────────────────┘ └──────────────────────────────┘ │
│                                                                    │
│ ┌──────────────────────────────────────────────────────────────┐ │
│ │ Recent Transactions                          [View All]      │ │
│ │                                                              │ │
│ │ Oct 27  WHOLE FOODS MARKET    -$87.43   Groceries           │ │
│ │ Oct 26  SALARY DEPOSIT       +$2,100    Income              │ │
│ │ Oct 25  NETFLIX.COM           -$14.99   Entertainment       │ │
│ │ Oct 24  SHELL GAS #1234       -$45.00   Transportation      │ │
│ │ Oct 23  STARBUCKS #456         -$5.25   Food & Dining       │ │
│ │                                                              │ │
│ └──────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

**Mobile Wireframe:**

```
┌──────────────────────────┐
│ Dashboard         ☰     │
├──────────────────────────┤
│ October 2024             │
│                          │
│ Income:  $4,200 ↑5%     │
│ Expense: $3,100 ↓2%     │
│ Net:     $1,100 ↑12%    │
├──────────────────────────┤
│ Top Spending:            │
│                          │
│ Rent      $1,200  39%    │
│ ██████████████████       │
│                          │
│ Groceries   $520  17%    │
│ ██████                   │
│                          │
│ Transport   $380  12%    │
│ ████                     │
│                          │
│ [View Details]           │
├──────────────────────────┤
│ Recent:                  │
│ ┌──────────────────────┐ │
│ │ WHOLE FOODS  -$87.43 │ │
│ │ SALARY      +$2,100  │ │
│ │ NETFLIX      -$14.99 │ │
│ └──────────────────────┘ │
│                          │
└──────────────────────────┘
```

**User Actions:**

1. **Change time period**
   - Click "[This Month ▼]"
   - Options: This Month, Last Month, Last 3 Months, Last 6 Months, This Year, Custom
   - Select "Last 3 Months" → All charts refresh

2. **Drill into category**
   - Click "Groceries $520" bar
   - Opens filtered transaction list showing only grocery transactions
   - Breadcrumb: Dashboard > Groceries > October 2024

3. **Export PDF**
   - Click [Export PDF]
   - Generates PDF snapshot of dashboard with all charts
   - Downloads: `finance-dashboard-october-2024.pdf`

4. **View all categories**
   - Click [View All Categories]
   - Opens full category breakdown with subcategories

---

## Navigation Flow

```
Home/Dashboard
  ├─ Upload Statement → Processing → Success → Dashboard
  ├─ Transactions
  │    └─ Click Transaction → Detail Modal
  │         ├─ Edit Category → Category Manager
  │         ├─ View Raw Data → Raw Data Tab
  │         └─ View Source PDF → PDF Viewer
  ├─ Categories → Category Manager
  │    └─ Create/Edit/Delete Categories
  └─ Settings
       ├─ Accounts
       ├─ Rules (categorization)
       └─ Profile
```

---

## Key UI Principles

**1. One-Click Actions**
- Upload should be one click, not a wizard
- Category changes save immediately (no "Save" button)
- Minimal confirmations (only for destructive actions)

**2. Contextual Actions**
- Actions appear where needed (edit button next to category)
- Hide advanced features until user needs them

**3. Progressive Disclosure**
- Show canonical data first (what user cares about)
- Raw data and decisions available via tabs (for power users)

**4. Responsive**
- Desktop: Table view (more data visible)
- Mobile: Card view (easier to scroll)
- Touch-friendly hit targets (44px minimum)

**5. Immediate Feedback**
- Changes appear instantly (optimistic UI)
- Background sync (don't block user)
- Toast notifications for confirmations

---

## Loading States

**Initial Page Load:**
```
┌──────────────────────────────────────┐
│ Loading transactions...              │
│ ▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░ 35%       │
└──────────────────────────────────────┘
```

**Skeleton Loaders (preferred over spinners):**
```
┌────────────────────────────────────────┐
│ ▓▓▓▓▓  ▓▓▓▓▓▓▓▓▓▓▓▓   ▓▓▓▓▓  ▓▓▓▓▓   │
│ ▓▓▓▓▓  ▓▓▓▓▓▓▓▓▓▓▓▓   ▓▓▓▓▓  ▓▓▓▓▓   │
│ ▓▓▓▓▓  ▓▓▓▓▓▓▓▓▓▓▓▓   ▓▓▓▓▓  ▓▓▓▓▓   │
└────────────────────────────────────────┘
```

---

## Empty States

**No Transactions:**
```
┌────────────────────────────────────────┐
│                                        │
│           📄                           │
│     No transactions yet                │
│                                        │
│  Upload your first bank statement      │
│  to get started.                       │
│                                        │
│       [Upload Statement]               │
│                                        │
└────────────────────────────────────────┘
```

**No Search Results:**
```
┌────────────────────────────────────────┐
│ No transactions match "starbuck"       │
│                                        │
│ Try:                                   │
│ • Checking your spelling               │
│ • Using different keywords             │
│ • Clearing filters                     │
│                                        │
│       [Reset Filters]                  │
└────────────────────────────────────────┘
```

---

## Summary

**This UI is:**
- ✅ Simple (5 main screens)
- ✅ Intuitive (standard patterns)
- ✅ Responsive (desktop + mobile)
- ✅ Fast (optimistic updates)

**This UI avoids:**
- ❌ Complex wizards (multi-step forms)
- ❌ Excessive modals (only for details)
- ❌ Hidden features (everything discoverable)
- ❌ Jargon (plain English)
