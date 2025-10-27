# 3. Categorize Spending

**Goal:** Organize transactions into spending categories

---

## What the user needs

**Automatic categorization:**
- System suggests category based on merchant name
- Examples:
  - "WHOLE FOODS" → Groceries
  - "SHELL GAS STATION" → Transportation
  - "NETFLIX" → Entertainment
- User can accept or change suggestion

**Manual categorization:**
- Click transaction → change category
- Dropdown with common categories:
  - Groceries
  - Rent/Mortgage
  - Utilities (electric, water, gas)
  - Transportation (gas, uber, parking)
  - Entertainment (netflix, movies, dining)
  - Healthcare (doctor, pharmacy)
  - Other

**Create custom categories:**
- Add new category with name
- Optional: set parent category (sub-categories)
- Example: "Coffee Shops" under "Food & Dining"

**Category rules:**
- "Always categorize STARBUCKS as Coffee Shops"
- System applies rule to future transactions
- Rules stored in simple config file (YAML)

---

## Scale expectations

- **Categories:** 20-30 total
- **Rules:** 50-100 merchant rules
- **Recategorization:** Instant (update one transaction)
- **Rule evaluation:** Under 100ms per transaction

---

## User experience

### Auto-categorization during import

```
Processing statement.pdf...
✓ 42 transactions imported
✓ 38 auto-categorized
⚠ 4 need manual review

[Review Uncategorized]
```

### Manual categorization

```
Transaction: ACME COFFEE SHOP - $4.75

Suggested category: [Uncategorized ▼]

Select category:
  ☐ Groceries
  ☐ Dining & Restaurants
  ☑ Coffee Shops
  ☐ Transportation
  ☐ Entertainment

☑ Create rule: Always categorize "ACME COFFEE" as Coffee Shops

[Save] [Cancel]
```

### Category rules (settings page)

```
┌─────────────────────────────────────────────────┐
│ Categorization Rules                   [+ New] │
├─────────────────────────────────────────────────┤
│ Merchant Pattern        → Category              │
├─────────────────────────────────────────────────┤
│ WHOLE FOODS            → Groceries              │
│ SAFEWAY                → Groceries              │
│ SHELL                  → Transportation         │
│ NETFLIX                → Entertainment          │
│ STARBUCKS              → Coffee Shops           │
│ ...                                             │
└─────────────────────────────────────────────────┘
```

---

## Technical constraints (simple)

**Rule storage:**
- YAML config file is sufficient
- Example: `categorization_rules.yaml`
- No need for database table initially

**Rule matching:**
- Simple substring matching (case-insensitive)
- Example: "whole foods" matches "WHOLE FOODS MARKET #1234"
- No need for regex or fuzzy matching initially

**Rule precedence:**
- More specific rules win
- "WHOLE FOODS MARKET" beats "WHOLE"
- Simple longest-match algorithm

**Categories:**
- Store in database table with parent_id (tree structure)
- Max 3 levels deep (Category > Subcategory > Sub-subcategory)

---

## What's NOT needed (out of scope)

❌ Machine learning for categorization
❌ Transaction splitting (one transaction, multiple categories)
❌ Percentage-based splits (60% groceries, 40% household)
❌ Merchant name normalization ("AMZN" → "Amazon")
❌ Category suggestions based on other users
❌ Import/export rules from other apps
❌ Bulk recategorization (recategorize 100 transactions at once)
❌ Category budgets (spending limits per category)
❌ Category forecasting (predict future spending)
