# Open Questions

> **Purpose:** Document undecided features and design choices that need user input

---

## Philosophy

**Not all decisions need to be made upfront.** Some features depend on actual usage patterns. This document tracks questions that should be answered based on real-world experience, not speculation.

**Decision Process:**
1. Build v1 without the feature
2. Use the app for 1-2 months
3. If pain point emerges → Revisit question
4. If no pain point → Feature not needed

---

## Question 1: Multi-Currency Support

**The Question:**
Should the app support transactions in multiple currencies (EUR, GBP, MXN, etc.)?

**Current State (v1):**
- All transactions assumed to be in USD
- No `currency_code` field
- No exchange rate tracking
- Foreign currency transactions show converted amount only

**Example Scenario:**
User travels to Europe. Credit card statement shows:
```
10/15/2024  RESTAURANT PARIS EUR 45.00
            EXCHANGE RATE 1.10              -$49.50
10/15/2024  FOREIGN TRANSACTION FEE          -$2.50
```

**Current handling:**
- Import as two USD transactions: -$49.50 and -$2.50
- Lose original EUR 45.00 information
- Can't calculate effective exchange rate including fee

**What multi-currency would enable:**
- Store original currency: `amount=45.00, currency=EUR`
- Store converted amount: `amount_usd=49.50`
- Track exchange rate: `rate=1.10`
- Calculate effective rate: `(49.50 + 2.50) / 45.00 = 1.156`
- View spending in original currency

**Decision Criteria:**

| If... | Then... |
|-------|---------|
| User rarely travels internationally (< 2x/year) | **Don't add** - Complexity not worth it |
| User travels frequently (> 4x/year) | **Add it** - Useful feature |
| User has foreign income/expenses | **Add it** - Essential |
| User only spends in USD | **Don't add** - Not needed |

**Data Model Impact:**
```sql
-- Would need to add:
ALTER TABLE transactions
ADD COLUMN currency_code TEXT DEFAULT 'USD';

ADD COLUMN amount_original REAL;  -- 45.00
ADD COLUMN currency_original TEXT;  -- 'EUR'
ADD COLUMN exchange_rate REAL;  -- 1.10

-- Or keep simple:
-- Just add currency_code, assume single currency per transaction
```

**Recommendation:**
**START WITHOUT IT.** Add only if user travels frequently.

---

## Question 2: Auto-Categorization Strategy

**The Question:**
Should categorization use:
- **Option A:** Simple pattern matching (YAML rules)
- **Option B:** Machine learning (train on user's history)
- **Option C:** Hybrid (rules + ML)

**Current State (v1):**
Using **Option A** - Pattern matching with YAML rules:
```yaml
- pattern: "WHOLE FOODS"
  category: "Groceries"
  confidence: 0.9
```

**Pros of Pattern Matching:**
- ✅ Simple to understand
- ✅ Deterministic (same input → same output)
- ✅ Easy to debug (just look at YAML file)
- ✅ User can edit rules directly
- ✅ No training data needed

**Cons of Pattern Matching:**
- ❌ Requires manual rule creation
- ❌ Can't learn from user corrections
- ❌ Doesn't adapt to new merchants
- ❌ 50-100 rules needed for good coverage

**What ML would enable:**
- Learn from corrections: User changes "Amazon" from "Shopping" to "Books" → System learns
- Adaptive: Accuracy improves over time
- Handle new merchants automatically

**What ML would require:**
- Training data: Need 100+ categorized transactions minimum
- Complexity: scikit-learn, model training, confidence scores
- Explainability: Harder to answer "why did you categorize this as X?"

**Decision Criteria:**

| If... | Then... |
|-------|---------|
| User has < 500 transactions | **Pattern matching** - Not enough training data |
| User is comfortable editing YAML | **Pattern matching** - User prefers control |
| Categorization accuracy > 85% with rules | **Pattern matching** - Good enough |
| User frequently corrects categories (> 10% of transactions) | **Try ML** - Rules not working |
| User wants "auto-improve" feature | **Try ML** - Sells the feature |

**Hybrid Approach (Option C):**
```python
def categorize_transaction(description):
    # Try pattern matching first (fast, explainable)
    category = apply_yaml_rules(description)
    if category and confidence > 0.8:
        return category

    # Fall back to ML for unknown merchants
    category = ml_model.predict(description)
    return category
```

**Recommendation:**
**START WITH RULES.** Measure correction rate. If > 10%, consider ML.

---

## Question 3: Split Transactions

**The Question:**
Should users be able to split a single transaction across multiple categories?

**Example Scenario:**
User shops at Target:
```
10/15/2024  TARGET #1234               -$127.50
```

But the $127.50 actually includes:
- Groceries: $45.00
- Household items: $35.50
- Clothing: $47.00

**Current State (v1):**
- Transaction assigned to ONE category only
- User can add note: "Includes groceries + household + clothing"
- No way to split into multiple categories

**What splitting would enable:**
- Accurate spending breakdown by category
- Tax deductible portions (if Target purchases include office supplies)
- Budget tracking per category

**What splitting would require:**
- UI for creating splits (% or $ amounts)
- Validation: Splits must sum to total
- Complex queries: "Show all grocery spending" → Include partial transactions
- Export complexity: Does split show as 1 row or 3 rows?

**Data Model Impact:**
```sql
-- Option 1: Split transactions table
CREATE TABLE transaction_splits (
  transaction_id TEXT,
  category_id TEXT,
  amount REAL,  -- Must sum to parent transaction amount
  percentage REAL  -- Optional: 35.4% for UI
);

-- Option 2: JSON field in transactions
ALTER TABLE transactions
ADD COLUMN splits JSON;
-- Example: [
--   {"category": "Groceries", "amount": 45.00},
--   {"category": "Household", "amount": 35.50},
--   {"category": "Clothing", "amount": 47.00}
-- ]
```

**Decision Criteria:**

| If... | Then... |
|-------|---------|
| User rarely shops at stores with multiple departments (< 5% of transactions) | **Don't add** - Edge case |
| User needs tax deduction tracking for mixed purchases | **Add it** - Required for taxes |
| User is comfortable with approximate category splits | **Don't add** - Notes field sufficient |
| Budget tracking requires exact splits | **Add it** - Affects budgets |

**Recommendation:**
**START WITHOUT IT.** Use notes field. Add if tax requirements emerge.

---

## Question 4: Receipt Attachments

**The Question:**
Should users be able to attach receipt images/PDFs to transactions?

**Example Scenario:**
- Business expense: Need receipt for reimbursement
- Large purchase: Warranty, return policy on receipt
- Tax deduction: IRS requires receipts for certain expenses

**Current State (v1):**
- No attachment support
- User can add note: "Receipt in Dropbox: /tax-docs/2024/target-receipt-oct15.pdf"

**What attachments would enable:**
- One-click receipt access from transaction
- Receipt storage integrated with app
- OCR to extract amount/date from receipt (validate against transaction)

**What attachments would require:**
- File upload UI (drag & drop images)
- Storage (500 receipts × 1 MB = 500 MB)
- Image viewer in transaction detail modal
- File management (delete, download)

**Data Model Impact:**
```sql
CREATE TABLE attachments (
  id TEXT PRIMARY KEY,
  transaction_id TEXT,  -- FK
  filename TEXT,
  file_path TEXT,  -- /uploads/receipts/abc123.jpg
  file_size INTEGER,
  mime_type TEXT,  -- image/jpeg, application/pdf
  uploaded_at DATETIME
);
```

**Decision Criteria:**

| If... | Then... |
|-------|---------|
| User is tracking personal expenses only | **Don't add** - Not needed |
| User needs receipts for business reimbursement | **Add it** - Essential for work |
| User needs receipts for tax audits | **Add it** - IRS compliance |
| User is comfortable storing receipts externally (Dropbox, Google Drive) | **Don't add** - External works |

**Recommendation:**
**START WITHOUT IT.** Add if business/tax use case emerges.

---

## Question 5: Budget Tracking

**The Question:**
Should the app include monthly budget tracking (planned vs. actual spending)?

**Example Scenario:**
User sets monthly budgets:
- Groceries: $500/month
- Dining: $200/month
- Entertainment: $100/month

App shows:
- Groceries: $520 spent (104% of budget) ⚠️
- Dining: $180 spent (90% of budget) ✅
- Entertainment: $85 spent (85% of budget) ✅

**Current State (v1):**
- No budget tracking
- User can see total spending by category in dashboard
- No alerts or warnings

**What budgets would enable:**
- Proactive alerts: "You've spent 90% of your Grocery budget"
- Goal setting: "Save $500 this month"
- Trend analysis: "Groceries over budget 3 months in a row"

**What budgets would require:**
- Budget setup UI (set amount per category)
- Budget progress bars in dashboard
- Alert system (email/push when over budget)
- Historical budget vs. actual comparison

**Data Model Impact:**
```sql
CREATE TABLE budgets (
  id TEXT PRIMARY KEY,
  category_id TEXT,
  amount REAL,  -- $500
  period TEXT,  -- 'monthly', 'weekly', 'yearly'
  start_date DATE,  -- 2024-10-01
  end_date DATE,  -- 2024-10-31
  alert_threshold REAL  -- 0.9 (alert at 90%)
);
```

**Decision Criteria:**

| If... | Then... |
|-------|---------|
| User wants passive tracking (just see spending) | **Don't add** - Dashboard sufficient |
| User wants active control (stay within budget) | **Add it** - Core feature |
| User has irregular income (freelancer, contractor) | **Maybe add** - Budgets less useful |
| User has fixed income and expenses | **Add it** - Very useful |

**Recommendation:**
**START WITHOUT IT.** Use app for 1-2 months. If user repeatedly checks "did I overspend on X?" → Add it.

---

## Question 6: Recurring Transaction Tracking

**The Question:**
What level of recurring transaction support?
- **Option A:** Manual (user creates series, links transactions manually)
- **Option B:** Auto-detect (system suggests series based on patterns)
- **Option C:** Proactive (system creates series automatically, user approves)

**Current State (v1):**
Implementing **Option B** - Auto-detect with suggestions

**How it works:**
- System detects: Netflix $14.99 every month on 15th
- User sees notification: "Looks like Netflix is a recurring payment - track it?"
- User approves → Series created
- Future Netflix charges auto-link to series

**Open Questions:**

**Q6.1: Missing Payment Alerts**
If November 15th passes without Netflix charge:
- **Alert immediately?** "Missing expected Netflix payment"
- **Wait 5 days?** (payment might be delayed)
- **User configurable?** Let user set grace period per series

**Decision:**
Wait 5 days by default. Add user setting if needed.

**Q6.2: Variance Tolerance**
Netflix charge is usually $14.99, but December shows $15.99:
- **Reject as non-match?** (create new transaction, don't link to series)
- **Accept with warning?** "Netflix charge was $1 higher - price increase?"
- **Auto-update series amount?** (assume new price going forward)

**Decision:**
Accept with warning. User reviews and either:
- Confirms price increase → Update series amount to $15.99
- Rejects → One-time variance, keep series at $14.99

**Q6.3: Series Cancellation**
User cancels Netflix. How should system handle?
- **Auto-detect?** If 2 months pass without charge, suggest "Mark series as cancelled?"
- **Manual only?** User must explicitly cancel series
- **Keep forever?** Series stays active indefinitely (alerts user every month)

**Decision:**
Auto-suggest after 2 missed payments. Don't auto-cancel (user might resume subscription).

---

## Question 7: Tax Preparation Integration

**The Question:**
How deep should tax support go?

**Levels of Support:**

**Level 1: Basic (v1)**
- Categorize transactions
- Tag tax-deductible expenses
- Export CSV for accountant

**Level 2: Intermediate**
- Pre-filled tax forms (Schedule C for self-employed)
- Multi-jurisdiction support (lived in 2 states)
- Mileage tracking (if car expenses deductible)

**Level 3: Advanced**
- Direct export to TurboTax, H&R Block
- Real-time estimated tax calculation
- Quarterly tax payment reminders
- Audit trail documentation

**Current State (v1):**
Implementing **Level 1** - Basic categorization

**Open Questions:**

**Q7.1: What's deductible?**
User marks transaction as "tax-deductible" but:
- Medical expenses only deductible if > 7.5% AGI
- Home office deduction has complex rules
- Meals are 50% deductible (for business)

**Should app:**
- **Trust user?** Let them mark anything deductible (accountant will verify)
- **Enforce IRS rules?** Block marking groceries as deductible
- **Warn user?** "Medical deductions require > $X total to claim"

**Decision:**
Trust user (v1). Add warnings if user requests them.

**Q7.2: Dual Purpose Expenses**
User buys laptop for 60% work, 40% personal:
- Split transaction? (see Question 3)
- Custom percentage field? `deductible_percentage=60%`
- Notes only? "60% work use"

**Decision:**
Notes only (v1). Add percentage field if common use case.

---

## Question 8: Mobile App vs. Web Only

**The Question:**
Should there be a native mobile app (iOS/Android)?

**Current State (v1):**
Web app only (responsive design works on mobile browser)

**What mobile app would enable:**
- **Receipt capture:** Take photo → auto-attach to transaction
- **Push notifications:** "Netflix payment due tomorrow"
- **Offline mode:** View transactions without internet
- **Fingerprint/Face ID:** Quick login

**What mobile app would require:**
- Native development (Swift for iOS, Kotlin for Android)
- App store submissions, reviews, updates
- Push notification infrastructure
- Higher complexity, more maintenance

**Decision Criteria:**

| If... | Then... |
|-------|---------|
| User primarily uses app on desktop | **Web only** - Sufficient |
| User wants receipt photo upload | **Build mobile** - Camera essential |
| User needs offline access | **Build mobile** - Web requires internet |
| User is comfortable with mobile web | **Web only** - Save development time |

**Recommendation:**
**WEB ONLY (v1).** Measure mobile web usage. If > 50% of traffic is mobile, consider native app.

---

## Question 9: Data Export Formats

**The Question:**
What export formats should be supported?

**Currently Supported:**
- CSV (for Excel, Google Sheets)

**Requested Formats:**

| Format | Use Case | Complexity |
|--------|----------|------------|
| Excel (XLSX) | Formatted reports with charts | Medium |
| PDF | Printable monthly summaries | Low |
| JSON | API integration, custom analysis | Low |
| QIF (Quicken) | Import into Quicken desktop | Medium |
| OFX (Open Financial Exchange) | Import into other finance software | High |

**Decision Criteria:**

| If... | Then... |
|-------|---------|
| User only needs data for spreadsheets | **CSV only** - Universal format |
| User wants to print monthly statements | **Add PDF** - Simple addition |
| User wants API access | **Add JSON** - Enables automation |
| User wants to migrate to other software | **Add OFX** - Standard format |

**Recommendation:**
CSV + PDF initially. Add others based on specific user requests.

---

## Question 10: Account Sync vs. Manual Upload

**The Question:**
Should the app support automatic bank sync (Plaid, Yodlee)?

**Current State (v1):**
Manual upload only (user downloads PDF from bank, uploads to app)

**What automatic sync would enable:**
- No manual uploads (transactions appear automatically)
- Real-time balance updates
- Multi-bank aggregation in one place

**What automatic sync would require:**
- Plaid integration ($0.25 per user per month)
- Security review (storing bank credentials)
- Bank connection maintenance (breaks when banks change APIs)
- Dealing with MFA challenges, captchas

**Why Manual Upload is Better (for v1):**
- ✅ **Privacy:** No bank credentials stored in app
- ✅ **Reliability:** PDF format rarely changes
- ✅ **Cost:** Free (no Plaid fees)
- ✅ **Control:** User decides when to import

**Why Automatic Sync is Better (for power users):**
- ✅ **Convenience:** Set it and forget it
- ✅ **Real-time:** See transactions immediately
- ✅ **Multi-bank:** Aggregate 10+ accounts easily

**Decision Criteria:**

| If... | Then... |
|-------|---------|
| User has 1-2 bank accounts | **Manual upload** - Once per month is fine |
| User has 10+ accounts | **Auto-sync** - Manual upload too tedious |
| User uploads < 12 times/year | **Manual upload** - Not burdensome |
| User wants daily balance checks | **Auto-sync** - Manual too slow |

**Recommendation:**
**MANUAL UPLOAD (v1).** Revisit if user complains about upload frequency.

---

## Summary: Decision Framework

**For each open question, apply this framework:**

1. **Build v1 without the feature**
2. **Instrument usage:** Track how often user encounters the limitation
3. **Set threshold:** If pain point occurs > X times/month → Add feature
4. **Validate with user:** Ask "Would feature Y solve problem Z?"
5. **Implement if yes, defer if no**

**Avoid:**
- ❌ Building features "just in case"
- ❌ Speculating about future needs
- ❌ Adding complexity without proven value

**Embrace:**
- ✅ Starting simple
- ✅ Learning from real usage
- ✅ Adding features when pain points emerge

**Key Principle:**
Every feature added makes the app slightly more complex. Complexity has a cost. Only add features when the benefit clearly outweighs the cost.
