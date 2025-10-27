# Core User Journeys

> **Purpose:** Tell the story of how users interact with the finance app through 11 critical flows

---

## Journey 1: When User Uploads Document

**The Sunday Morning Ritual**

Darwin opens the finance app on Sunday morning with his Bank of America statement from last month. He's ready to review his spending.

**The Upload Experience:**

1. **Finding the file**
   - Clicks "Upload Statement" button
   - File picker opens
   - Navigates to ~/Downloads/bofa-statement-october.pdf
   - Selects the file

2. **The system accepts it immediately**
   - No dropdown asking "what type of file is this?"
   - No confirmation dialog
   - Progress bar appears: "Uploading... 1.2 MB"
   - Upload completes in 2 seconds

3. **Processing begins**
   - Screen shows: "Processing statement... this may take 30 seconds"
   - User can navigate away - processing happens in background
   - Notification will appear when complete

**Why this matters:**
- The user shouldn't need to specify "source_type=bofa_pdf" - the system should detect this
- But for v1, we'll ask because PDF format detection is complex
- The upload should feel instant even though parsing takes 30 seconds

**What can go wrong:**
- File is not a PDF → Show error: "Please upload a PDF file"
- File is corrupted → Show error: "Could not read PDF - file may be corrupted"
- File was already uploaded → Show warning: "This file was uploaded on Oct 15"

---

## Journey 2: When System Does Parsing

**Behind the Scenes (User doesn't see this)**

The PDF is in storage. A worker picks up the job.

**The Parsing Flow:**

1. **Opening the PDF**
   - PyPDF2 library opens the file
   - Scans for text content
   - Identifies table structure on page 2

2. **Extracting transactions**
   - Finds the transaction table header: "Date | Description | Amount"
   - Reads each row
   - Extracts 42 transactions
   - Raw data: dates as "10/15/2024", amounts as "$87.43", descriptions as "WHOLE FOODS MARKET #1234"

3. **Saving raw observations**
   - Each transaction saved as "observation" (raw, unprocessed)
   - Fields: date_raw="10/15/2024", description_raw="WHOLE FOODS MARKET #1234", amount_raw="$87.43"
   - Status changes: queued_for_parse → parsing → parsed

**The Critical Decision:**
- Store data AS-IS, don't clean it yet
- Why? If we discover the cleaner has a bug, we can re-process from raw data
- The raw observations are immutable - never modified

**Edge Cases:**
- PDF has 0 transactions → Status: parsed (with warning, not error)
- PDF has 1000+ transactions → Works fine, just takes longer
- PDF format changed (BoFA redesigned their statement) → Parser fails, user gets clear error

---

## Journey 3: When User Navigates OL (Objective Layer)

**The Drill-Down Experience**

Darwin sees his transaction list. One transaction catches his eye:

```
Oct 15    WHOLE FOODS MARKET    -$87.43    Groceries
```

**He clicks on it. A modal opens showing:**

**Tab 1: Canonical Data (What we determined is true)**
```
Date:         October 15, 2024
Merchant:     Whole Foods Market
Amount:       -87.43 USD
Category:     Groceries
Account:      Chase Checking (****1234)
Confidence:   High (0.95)
```

**Tab 2: Raw Observation (What the PDF said)**
```
Date:         "10/15/2024"
Description:  "WHOLE FOODS MARKET #1234 SAN FRANCISCO CA"
Amount:       "$87.43"
Account:      "****1234"
```

**Tab 3: Normalization Decisions (How we got from raw → canonical)**
```
Date Cleaning:
  Raw: "10/15/2024" → Canonical: 2024-10-15 (ISO 8601)
  Rule: Parse MM/DD/YYYY format

Merchant Normalization:
  Raw: "WHOLE FOODS MARKET #1234 SAN FRANCISCO CA"
  → Canonical: "Whole Foods Market"
  Rule: Strip store number and location

Amount Parsing:
  Raw: "$87.43" → Canonical: -87.43
  Rule: Remove "$", negate (it's an expense)

Category Assignment:
  Merchant: "Whole Foods Market" → Category: "Groceries"
  Rule: Merchant pattern match (configured in YAML)
```

**Tab 4: Source Document**
- [View PDF] button → Opens bofa-statement-october.pdf at page 2
- Transaction highlighted in yellow

**Why this matters:**
- Full transparency - user can see exactly how we processed their data
- If something looks wrong, they can see where it came from
- Builds trust

---

## Journey 4: When User Navigates RL (Representation Layer)

**The Dashboard View**

Darwin wants to see his spending trends. He navigates to the Dashboard.

**What he sees:**

```
┌─────────────────────────────────────────────────┐
│ October 2024 Summary                            │
├─────────────────────────────────────────────────┤
│ Income:      $4,200  ↑ 5% vs Sept               │
│ Expenses:    $3,100  ↓ 2% vs Sept               │
│ Net Savings: $1,100  ↑ 12% vs Sept              │
├─────────────────────────────────────────────────┤
│ Top Spending Categories:                        │
│                                                 │
│ 🏠 Rent           $1,200  ████████████ (39%)   │
│ 🍔 Groceries        $520  ████         (17%)   │
│ 🚗 Transportation   $380  ███          (12%)   │
│ ⚡ Utilities        $250  ██           (8%)    │
│ 🎬 Entertainment    $180  █            (6%)    │
└─────────────────────────────────────────────────┘
```

**Interactive Elements:**
- Click "Rent" → Drills down to all rent transactions
- Click "Groceries" → Shows breakdown by store (Whole Foods, Safeway, Trader Joe's)
- Date picker → Change month being viewed

**Why this is RL (Representation Layer):**
- It's a VIEW built on top of OL (Objective Layer) data
- The canonical transactions haven't changed
- We're just presenting them differently
- Same data, different perspective

---

## Journey 5: When System Detects Duplicate (Apple Card)

**The Deduplication Problem**

Darwin uploads his October statement. The system starts processing.

**What happens:**

1. **System reads transaction:** "APPLE.COM/BILL $14.99" on Oct 15
2. **Hash calculation:** Computes hash of (date + merchant + amount + account)
3. **Duplicate check:** Searches for existing transaction with same hash
4. **Match found:** Identical transaction from Oct 15 already exists (uploaded 2 weeks ago from different PDF)

**System Decision:**
```
Status: DUPLICATE DETECTED
Action: Skip creating canonical transaction
Reason: Hash collision - same transaction already in system
Original: TX_abc123 (uploaded Oct 16 from statement-september.pdf)
Duplicate: TX_xyz789 (current upload from statement-october.pdf)
```

**What user sees:**
- Import summary: "42 transactions processed, 1 duplicate skipped"
- Details available if they click "Show duplicates"

**Why this matters:**
- Banks sometimes include previous month's transactions on new statements
- Credit cards show pending transactions on multiple statements
- Without deduplication, user would see duplicate charges

**The Edge Case:**
What if two identical charges legitimately happened on same day?
- Example: Two $4.75 Starbucks purchases on Oct 15
- Hash would be identical
- Current solution: User can manually "un-merge" if needed
- Future: Use transaction time (HH:MM:SS) if available

---

## Journey 6: When System Does Account Resolution

**The Account Matching Problem**

The PDF says: "Account: ****1234"

But which account is that? Darwin has 6 accounts.

**Resolution Flow:**

1. **System reads raw observation:**
   - Account identifier: "****1234"
   - From PDF: bofa-statement-october.pdf

2. **Lookup in Account Registry:**
   ```
   Registered Accounts:
   - Chase Checking (****1234)     ← MATCH!
   - Chase Savings (****5678)
   - BoFA Checking (****9012)
   - BoFA Credit Card (****3456)
   - Apple Card (****7890)
   - Amex (****2345)
   ```

3. **Match found:**
   - Canonical transaction gets: account_id = "ACC_chase_checking"
   - Link established: Transaction → Account

**What if no match?**
- Status: UNRESOLVED
- User sees: "Unknown account ****1234 - please map to existing account"
- User creates mapping: "****1234" → "Chase Checking"
- System re-processes: All transactions with ****1234 now linked correctly

**Why Account Registry is closed:**
- User explicitly creates accounts (not auto-generated from statements)
- Prevents "account explosion" (100 accounts from typos)
- Ensures clean data

---

## Journey 7: When System Does Counterparty Resolution

**The Merchant Matching Problem**

Raw description: "WHOLE FOODS MARKET #1234 SAN FRANCISCO CA"

Who is the counterparty? What's the canonical merchant name?

**Resolution Flow:**

1. **System reads raw description:**
   - "WHOLE FOODS MARKET #1234 SAN FRANCISCO CA"

2. **Normalization Rules Applied:**
   ```yaml
   # merchantnormalization_rules.yaml
   - pattern: "WHOLE FOODS MARKET #\d+"
     canonical: "Whole Foods Market"
     strip: ["#\d+", "SAN FRANCISCO CA"]
   ```

3. **Counterparty Lookup:**
   - Search existing counterparties for "Whole Foods Market"
   - **Match found:** counterparty_id = "CP_wholefoods"

4. **If no match → Create new counterparty:**
   ```
   Canonical Name: "Whole Foods Market"
   Aliases: ["WHOLE FOODS MARKET #1234", "WFM", "WHOLE FOODS"]
   Type: merchant
   Category: groceries
   ```

**Why Counterparty Registry is open:**
- Auto-creates from merchant names
- Learns aliases over time
- User can merge duplicates later

**The Fuzzy Matching Example:**
- Transaction 1: "WHOLE FOODS MARKET #1234"
- Transaction 2: "WHOLE FOODS MKT #5678"
- Transaction 3: "WFM SAN FRANCISCO"

System creates 3 counterparties initially. User sees duplicate suggestion:
"These look similar - merge them?"
→ User confirms → System merges into one canonical counterparty

---

## Journey 8: When System Does Clustering (Post-Entity Resolution)

**The Pattern Discovery Problem**

After account and counterparty resolution, system looks for patterns.

**Clustering Flow:**

1. **Find similar transactions:**
   ```
   Group 1: Starbucks Purchases
   - Oct 1:  STARBUCKS #123 SF    $4.75
   - Oct 5:  STARBUCKS #456 OAK   $5.25
   - Oct 12: STARBUCKS #123 SF    $4.75
   - Oct 20: STARBUCKS #789 NYC   $6.50

   Common features:
   - Counterparty: Starbucks
   - Category: Coffee
   - Amount range: $4-7
   - Frequency: Weekly
   ```

2. **System suggests:**
   ```
   Cluster: "Weekly Coffee"
   Members: 4 transactions
   Average: $5.31
   Confidence: High (0.89)
   ```

3. **User can:**
   - Accept cluster → Transactions get tag "weekly-coffee"
   - Reject cluster → System won't suggest again
   - Modify cluster → Add/remove transactions

**Why clustering matters:**
- Discovers spending patterns automatically
- Helps user understand habits
- Enables budgeting by pattern

**The Threshold:**
- Minimum 3 transactions to form cluster
- Similarity score > 0.7 (configurable)
- Time window: last 90 days

---

## Journey 9: When User Corrects an Error

**The Mistake Discovery**

Darwin notices: "AMZN MKTPLACE" on Oct 10 for $45.99 is categorized as "Shopping"

But it was actually a book purchase - should be "Books & Education"

**Correction Flow:**

1. **User clicks transaction**
2. **Clicks "Edit Category"**
3. **Dropdown shows:**
   ```
   Current: Shopping (from auto-categorization rule)

   Change to:
   ● Books & Education
   ○ Shopping
   ○ Entertainment
   ○ Other
   ```

4. **User selects "Books & Education" and clicks Save**

**What happens behind the scenes:**

```
1. System creates override:
   {
     transaction_id: "TX_abc123",
     field: "category",
     old_value: "Shopping",
     new_value: "Books & Education",
     reason: "manual_correction",
     corrected_by: "darwin",
     corrected_at: "2024-10-27T10:30:00Z"
   }

2. Canonical transaction updated:
   category = "Books & Education"

3. Provenance entry created:
   "Transaction TX_abc123 category changed from Shopping to Books & Education
    by user darwin at 2024-10-27T10:30:00Z"
```

**User sees:**
- Transaction now shows "Books & Education"
- Small indicator: "Edited" (with tooltip showing original value)

**Can corrections be reverted?**
- Yes! User clicks "View History" → Sees all changes → "Revert to Original"
- System restores "Shopping" and logs the revert

**Why this matters:**
- Auto-categorization isn't perfect
- User should be able to fix mistakes
- Full audit trail of changes

---

## Journey 10: When System Connects Transaction to Series

**The Recurring Payment Pattern**

Darwin has Netflix: $14.99 every month on the 15th.

**Pattern Detection:**

1. **System sees transactions:**
   ```
   Sep 15: NETFLIX.COM    $14.99
   Oct 15: NETFLIX.COM    $14.99
   Nov 15: NETFLIX.COM    $14.99
   ```

2. **Pattern recognized:**
   ```
   Series Detected:
   - Name: Netflix Subscription
   - Amount: $14.99 (± $1 tolerance)
   - Frequency: Monthly
   - Day: 15th of month
   - Counterparty: Netflix
   - Confidence: High (0.95)
   ```

3. **System creates Series:**
   ```
   series_id: "SER_netflix"
   template: {
     amount: 14.99,
     counterparty: "Netflix",
     category: "Entertainment",
     recurrence: "monthly"
   }
   ```

4. **Links transactions to series:**
   ```
   TX_sep15 → SER_netflix (instance 1)
   TX_oct15 → SER_netflix (instance 2)
   TX_nov15 → SER_netflix (instance 3)
   ```

**User Experience:**
- Transaction shows badge: "Recurring: Netflix Subscription"
- Dashboard shows: "Monthly Subscriptions: $89.94" (sum of all recurring)
- Can click to see all instances

**Variance Detection:**
- If December charge is $15.99 instead of $14.99
- System flags: "Netflix charge was $1 higher than usual - price increase?"
- User can accept new amount or investigate

**Missing Payment Detection:**
- If January 15th passes without Netflix charge
- System alerts: "Missing expected Netflix payment"
- User can mark as "cancelled subscription" or "still pending"

---

## Journey 11: When System Categorizes for USA Taxes

**The Tax Season Preparation**

April 1st. Darwin needs to prepare his taxes.

**Tax Categorization:**

1. **System has transactions categorized:**
   ```
   Groceries: $6,240
   Rent: $14,400
   Utilities: $3,000
   Healthcare: $2,500
   Charitable Donations: $1,200
   ```

2. **Tax category mapping applied:**
   ```yaml
   # USA Tax Rules
   Deductible:
     - Medical Expenses: Healthcare ($2,500)
     - Charitable: Charitable Donations ($1,200)

   Non-Deductible:
     - Personal: Groceries, Rent, Utilities ($23,640)
   ```

3. **Tax report generated:**
   ```
   Schedule A (Itemized Deductions):
   - Medical: $2,500
   - Charitable: $1,200
   Total Itemized: $3,700

   Standard Deduction: $13,850
   Recommendation: Take standard deduction
   ```

**Why category matters for taxes:**
- Medical expenses only deductible if > 7.5% AGI
- Charitable donations require receipts
- Business expenses (if self-employed) fully deductible

**The Multi-Jurisdiction Problem:**
- Darwin lives in California but worked remotely from Mexico for 3 months
- Some transactions in MXN (Mexican Pesos)
- Needs to track which income/expenses apply to which jurisdiction

**Solution:**
- Transactions get jurisdiction tags: "USA" or "Mexico"
- Separate tax reports per jurisdiction
- FX conversion applied at transaction date rates

---

## Journey 12: When System Links FX Fee Transaction

**The Foreign Exchange Hidden Fee**

Darwin made a purchase in Europe:

```
Oct 10: RESTAURANT PARIS    -€45.00
Oct 10: FOREIGN FX FEE      -$2.50
```

**The Connection:**

1. **System sees two transactions same day:**
   - Transaction 1: €45.00 (converted to $49.50 at rate 1.10)
   - Transaction 2: $2.50 (FX fee)

2. **Pattern match:**
   ```
   Criteria for FX fee link:
   - Same date (or next day)
   - Description contains "FX FEE", "FOREIGN", "EXCHANGE"
   - Amount < 5% of foreign transaction
   - Account: Same credit card
   ```

3. **Relationship created:**
   ```
   type: "foreign_exchange_fee"
   primary: TX_paris (€45.00)
   fee: TX_fxfee ($2.50)
   total_cost: $52.00
   effective_rate: 1.156 (including fee)
   ```

**User Experience:**
- Primary transaction shows: "€45.00 ($49.50 + $2.50 FX fee = $52.00 total)"
- FX fee transaction shows: "Fee for: Restaurant Paris transaction"
- Both linked visually in UI

**Why this matters:**
- Credit card companies hide FX fees as separate transactions
- User needs to see true cost of international purchases
- Affects spending analysis accuracy

**Edge Case - Multi-Currency Travel:**
Darwin travels to Europe for 2 weeks. 30 transactions in EUR, 5 FX fees.

System must:
1. Match each FX fee to correct transaction (by amount heuristic)
2. Handle split fees (one FX fee for multiple small transactions)
3. Allow manual linking if auto-match fails

---

## Summary: The User Journey Map

```
1. Upload Document → System stores, queues for parsing
2. System Parses → Extracts raw observations
3. User Navigates OL → Full transparency into processing
4. User Navigates RL → Insights and visualizations
5. Duplicate Detection → Prevents double-counting
6. Account Resolution → Links to user's accounts
7. Counterparty Resolution → Identifies merchants
8. Clustering → Discovers spending patterns
9. User Correction → Fixes categorization errors
10. Series Connection → Identifies recurring payments
11. Tax Categorization → Prepares for tax season
12. FX Fee Linking → Shows true cost of international purchases
```

**Key Principle:**
The system should be intelligent but transparent. Every decision should be explainable and correctable by the user.
