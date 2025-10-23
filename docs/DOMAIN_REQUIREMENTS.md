# Finance Domain Requirements - User Context

> **Purpose:** Domain-specific requirements extracted from user interviews. This document informs vertical design decisions and ensures the finance app implementation matches real-world usage patterns.
>
> **Source:** 41-question questionnaire completed 2025-05-23
>
> **Status:** Reference document for all verticals

---

## 📊 User Profile

**Tax Situation:**
- Dual nationality (USA + Mexico)
- Fiscal resident: Mexico
- USA taxes: Federal + Self-Employment Tax
- No LLC/S-Corp (personal account with SSN)
- Self-filing (no accountant with direct access)

**Business Model:**
- Freelance consultant/contractor
- Currently: 0 active clients
- Historical: ~10 clients total
- Payment mix: ~90% direct, ~10% via Stripe
- Recurring revenue model (monthly retainers)

---

## 💳 Account Structure

### Primary Accounts (6 active)

| Account | Type | Currency | Purpose | Monthly Volume |
|---------|------|----------|---------|----------------|
| **BofA Debit** | Checking | USD | **Hub central** - Recibe ingresos, distribuye | ~20 transactions |
| **BofA Credit** | Credit Card | USD | Compras grandes, viajes | ~0 (no visible) |
| **Apple Card** | Credit Card | USD | Compras diarias, cash advances | ~100 transactions |
| **Scotia Debit** | Checking | MXN | Gastos en México | ~60 transactions |
| **Wise** | Transfer Service | USD/MXN | Conversión USD→MXN, pagos contractors | ~15 transactions |
| **Stripe** | Payment Processor | USD | Recibe pagos de ~1 cliente de 10 | ~21 balance entries |

**Total transactions/month:** ~195

---

## 💰 Money Flow Patterns

### Inbound (Income)

**Route 1: Direct to BofA** (Primary - ~90%)
```
Client → BofA Debit
Examples:
- HubSpot via Coupa Pay: $5,000/month (was recurring)
- Bloom Financial via Wise: $2,000/month (was recurring)
```

**Route 2: Via Stripe** (~10%)
```
Client → Stripe (-2.9% - $0.30) → BofA Debit
```

**Historical income:** ~$9,000/month average

---

### Outbound (Distribution)

**From BofA Debit:**

**1. To Wise → Scotia** (USD to MXN conversion)
```
BofA → Wise → Scotia
Frequency: 3-4 times/month
Amounts: $300-$3,000 per transfer
Purpose: Fund Mexico expenses
```

**2. To Credit Cards** (Payment)
```
BofA → BofA Credit: ~$1,200/month
BofA → Apple Card: ~$1,835/month
```

**3. Direct Expenses**
```
Software subscriptions: ~$540/month
Other: varies
```

**From Scotia:**
```
SPEI to individuals: 10-15/month
Utilities: CFE, SAT
Daily expenses: restaurants, groceries, uber
```

---

## 🔄 Transfer Reconciliation Requirements

**Critical pain point:** "No sé en qué cuenta buscar" (I don't know which account to search)

**Auto-matching needed for:**

1. **BofA → Wise → Scotia chain**
   - Challenge: No common transaction ID between platforms
   - BofA shows: "Wise Inc DES:WISE ID:TrnWise -$2,000"
   - Wise shows: "Sent | Eugenio Castro Garza | MXN amount + USD fee"
   - Scotia shows: "TRANSF SPEI - WISE PAYMENTS LIMITED +$X MXN"
   - **Matching strategy:** Amount (±$1 USD) + Date (±2 days) + Direction

2. **Stripe → BofA**
   - Currently: User does NOT cross-reference manually
   - Should: Link Stripe payout to BofA deposit
   - Purpose: Track fee costs, match to client invoice

**Frequency:** 10-15 inter-account transfers/month

---

## 🛒 Expense Categories & Tax Treatment

### Business (Deductible)

**Software & SaaS** (~$540/month - 100% deductible)
- OpenAI ChatGPT: $200
- Twitter/X Dev: $193.65 + $41.29
- Figma: $23.20
- Perplexity: $20
- ExpressVPN: $12.95
- Spotify: $11.99
- Slack: $10.15
- Others: ~$27

**Contractors** (~$2,000+/month - 100% deductible)
- Democratize Knowledge LLC (Shin): $2,000/month
- manuel maria mao: $100-200/project
- Maximiliano Takeda: ~$2,000 MXN variable

**Payment Processing Fees** (~$150/month - 100% deductible)
- Stripe: ~3% on transactions
- Foreign transaction fees: $95.09/month (Apple Card)
- Wire fees: varies

---

### Personal (Non-deductible)

**Food** (~$850/month)
- Groceries: $200-400/month (Supercenter, GNC)
- Restaurants: $300-500/month
- Delivery (Uber Eats): $150/month

**Transportation** (~$200/month)
- Uber: $50-150/month
- Gas: $50-100/month
- Occasional flights

**Utilities** (~$200/month MXN)
- CFE (Electricity): ~$356 MXN
- Internet: [not visible]
- Phone: [not visible]

**Subscriptions Personal** (~$50/month)
- YouTube Premium: 139 MXN (~$7)
- Google One: 395 MXN (~$20)
- Apple Services: $9.99-$44.97 (multiple)
- Uber One: $3.57

**Taxes**
- SAT (Mexico): ~$3,100 MXN/month variable

---

### Partially Deductible (Requires Context)

**Meals with Clients** (50% USA, 100% Mexico if factura)
- Need: "Note" field to mark business vs personal
- Example: Cocobongo $97 - was this client meeting or personal?

**Travel** (100% if business purpose documented)
- Need: Trip purpose tracking
- Example: Airbnb $144.74, Aeromexico $231.78

**Home Office** (partial in USA)
- Internet, electricity - need % business use
- Currently: Not tracking separately

---

## 🔁 Recurring Payments (Series Detection)

### Fixed Amount Subscriptions (15 total)

| Service | Amount | Account | Frequency | Alert Priority |
|---------|--------|---------|-----------|----------------|
| OpenAI | $200 | BofA Credit | Monthly | 🔴 Critical (API dependency) |
| Twitter/X | $193.65 + $41.29 | BofA Credit/Debit | Monthly | 🟡 Medium |
| Figma | $23.20 | BofA Credit | Monthly | 🟡 Medium |
| Perplexity | $20 | BofA Credit | Monthly | 🟢 Low |
| Others | ~$150 total | Various | Monthly | 🟢 Low |

**User needs:**
- ✅ Alert if subscription didn't charge (API might be down)
- ✅ Alert if price changed (budget tracking)
- ✅ List of unused subscriptions (cancel waste)

---

### Variable Amount Recurring

| Category | Variability | Range Observed |
|----------|-------------|----------------|
| CFE (Electricity) | Consumption-based | $356 MXN |
| SAT (Taxes) | Income-based | $3,100-$3,500 MXN |
| Uber | Usage-based | $49-$401 MXN/ride |
| Uber Eats | Order-based | $640-$1,365 MXN/order |
| Groceries | Shopping-based | $214-$2,303 MXN |

**User needs:**
- ✅ Detect anomalies (CFE suddenly $800 = something wrong)
- ✅ Average vs actual comparison
- ✅ Budget vs actual alerts

---

## 📊 Reporting Requirements

### Critical Questions User Needs Answered

**Income Analysis:**
1. "¿Cuánto ganó mi negocio (ingresos - gastos) en Q1?"
2. "¿A qué cliente le facturé más en 2024?" (auto-detect from deposits)
3. "¿Qué % de mis ingresos se va a fees?" (Stripe + foreign transaction)

**Expense Analysis:**
4. "¿Cuánto gasté en comida este mes?" (across 3 accounts)
5. "¿Cuánto gasto promedio en subscriptions?" (detect all recurring)
6. "¿Cuánto gasté en software deducible?" (for taxes)
7. "¿México vs USA expenses este trimestre?"

**Cash Flow:**
8. "¿Cuánto dinero está 'en tránsito' ahora?" (BofA→Wise→Scotia pending)
9. "¿Cuánto cash disponible REAL tengo?" (exclude pending)
10. "¿Mi burn rate aumentó vs promedio?" (early warning)

**Tax Prep:**
11. "Export CSV para contador" (all deductible expenses)
12. "¿Cuánto perdí en foreign transaction fees?" (deductible)
13. "¿A quién le pagué más como contractor?" (1099 tracking)

---

### Dashboard vs Reports

**Real-time Dashboard (Daily use):**
- Current cash available (all accounts)
- Transfers in-transit status
- Alerts (low balance, payment due, anomalies)
- MTD income, expenses, net

**Monthly Reports (Tax/Analysis):**
- Spending by category breakdown
- Income by client (auto-detected)
- Deductible vs non-deductible totals
- Export: CSV for CPA, PDF for records

**Comparisons:**
- Month vs month (detect trends)
- Year vs year (growth tracking)
- Actual vs budget (if budgets defined)

---

## 🔧 Client/Counterparty Auto-Detection

**Challenge:** User has 0 current clients but historical data will have ~10

**Detection strategy needed:**

**Inbound (Clients):**
```
Pattern: "WISE US INC - From [ClientName] Via WISE"
Examples:
- "From Bloom Financial Corp. Via WISE" → Client: Bloom Financial
- "HubSpot Inc - Coupa Pay 286-31598" → Client: HubSpot
```

**Detection rules:**
1. Recurring deposits (same source, similar amount, monthly)
2. Merchant pattern matching ("From X Via WISE" = client X)
3. Amount threshold (>$500 likely client, not refund)

**Outbound (Contractors):**
```
Pattern: Wise "Sent | [PersonName]"
Examples:
- "Sent | manuel maria mao" → Contractor
- "Sent | Eugenio Castro Garza" → Self (inter-account)
```

**Detection rules:**
1. Recurring Wise outbound payments
2. Amount threshold ($100-$5,000 = likely contractor)
3. Reference notes ("Doc Research", "April salary")

---

## ⚠️ Error Patterns Observed

### 1. Duplicate Charges (Most Common)

**Evidence from Scotia statement:**
```
19 FEB | ST UBER EATS: -$640.98
19 FEB | ST UBER EATS: -$640.98 (duplicate)
19 FEB | REV.ST UBER EATS: +$640.98 (reversed)
```

**Frequency:** 5-8 reversals per month observed

**User needs:**
- Auto-detect duplicates (same merchant, amount, date)
- Flag for review before categorizing
- Track reversal patterns (is Uber Eats buggy?)

---

### 2. Merchant Name Inconsistencies

**Examples from statements:**
| Statement Shows | Actual Merchant |
|-----------------|-----------------|
| "ST UBER CARG RECUR" | Uber |
| "STRIPE UBER TRIP CIU" | Uber (via Stripe) |
| "MERPAGO*COCOBONGO" | Cocobongo (via MercadoPago) |
| "CLIP MX*HANAICHI" | Hanaichi (via Clip) |
| "OPLINEA*SONAMBULOCAFE" | Sonambulocafe |

**User needs:**
- Normalize merchant names (3.8 Cluster Rules)
- Learn from manual corrections
- Confidence scoring when ambiguous

---

### 3. Foreign Transaction Fees (Unexpected)

**Apple Card charges 3% FX fee on MXN purchases:**
```
04/26 | MERPAGO*COCOBONGO | $97.25
04/26 | FOREIGN TRANSACTION FEE | $2.91 (3%)
```

**Total fees observed:** $95.09 in one month

**User needs:**
- Auto-link fee to original transaction
- Track total FX fees (deductible business expense)
- Alert on high-fee transactions (use Scotia instead?)

---

## 🎯 Feature Priority (From Pain Points)

### 🔴 **P0 - Critical (MVP)**

1. **Universal search across all accounts**
   - Pain point: "No sé en qué cuenta buscar"
   - Impact: 2+ hours/month wasted searching

2. **Transfer auto-matching (BofA→Wise→Scotia)**
   - Pain point: "Dinero en limbo"
   - Impact: Cash flow uncertainty

3. **Fee tracking & net income calculation**
   - Pain point: "No sé cuánto realmente gané"
   - Impact: $300+/month hidden costs

4. **Auto-categorization (learn from past)**
   - Pain point: "Me toma 2 horas categorizar"
   - Impact: 24 hours/year wasted

---

### 🟡 **P1 - Important (Post-MVP)**

5. **Recurring payment detection & alerts**
   - Alert if expected payment didn't occur
   - Alert if price changed

6. **Deductible vs non-deductible tagging**
   - For tax preparation
   - Dual jurisdiction (USA + Mexico)

7. **Client/contractor auto-detection**
   - From deposit patterns
   - For 1099 tracking

8. **Duplicate detection & reversal tracking**

---

### 🟢 **P2 - Nice-to-have (Future)**

9. **Receipt photo upload + OCR**
10. **Forecast / projections**
11. **Saved views / custom dashboards**
12. **Contador read-only access**

---

## 💡 Insights for Primitive Design

### Multi-Currency is NOT Optional

- Every transaction has original currency + converted amount
- Exchange rates vary (Wise rate ≠ bank rate)
- Need to store: `amount_original`, `currency_original`, `amount_usd`, `fx_rate`, `fx_fee`

---

### State ≠ Category

**State:** System-driven (parsed, normalized, reconciled)
**Category:** User/rule-driven (Software, Food, Travel)

Both need to be tracked independently.

---

### Confidence Scores are Critical

Many transactions are ambiguous:
- "J. Smith" = John or Jane?
- "Uber Eats" = Personal or business meal with client?
- Transfer BofA→Wise $2,000 = which of 3 Wise transactions?

**All automated decisions need confidence_score** (0.0-1.0)

**Thresholds:**
- > 0.95: Auto-accept
- 0.70-0.95: Suggest (user confirms)
- < 0.70: Manual review required

---

### Provenance is Non-Negotiable

User does corrections frequently:
- Recategorize transaction
- Mark as business vs personal
- Link/unlink transfers

**Every correction must flow through ProvenanceLedger:**
- Who changed it
- When
- Why (reason field)
- What was old value
- What is new value

---

## 🔍 Transaction Examples (Real Data)

### Typical BofA Debit Month

**Income:**
```
04/15 | WISE US INC - From Bloom Financial Corp. | +$2,000
04/30 | HubSpot Inc - Coupa Pay 286-31598 | +$5,000
04/30 | WISE US INC - From Bloom Financial Corp. | +$2,000
Total: +$9,000
```

**Outbound Transfers:**
```
04/23 | Wise Inc DES:WISE | -$2,000
04/30 | Wise Inc DES:WISE | -$3,000
05/06 | Wise Inc DES:WISE | -$500
Total: -$5,500
```

**Bill Payments:**
```
04/23 | Bank of America Credit Card Bill Payment | -$843.62
04/24 | Bank of America Credit Card Bill Payment | -$174.38
05/01 | APPLECARD GSBANK Payment | -$1,835.11
Total: -$2,853.11
```

**Subscriptions:**
```
05/02 | Slack | -$10.15
05/02 | Uber One | -$3.57
05/08 | X/Twitter | -$41.29
Total: -$55.01
```

**Net month:** ~+$590

---

### Typical Scotia Debit Month (MXN)

**Income (All from Wise):**
```
20 FEB | TRANSF SPEI - WISE PAYMENTS | +$20,137.88
27 FEB | TRANSF SPEI - WISE PAYMENTS | +$10,168.10
06 MAR | TRANSF SPEI - WISE PAYMENTS | +$51,631.07
Total: ~$82,000 MXN (~$4,200 USD)
```

**Expenses:**
```
Food: ~$5,000 MXN
Utilities: ~$4,500 MXN (SAT, CFE, etc.)
Transport: ~$3,000 MXN (Uber, gas)
Transfers to people: ~$40,000 MXN
Other: ~$10,000 MXN
Total: ~$62,500 MXN
```

**Pattern:** Receive bulk transfers from Wise, spend locally in MXN

---

## 📋 Checklist for Vertical Design

When designing any vertical that touches user data, verify:

- [ ] **Multi-account aware** - Can search/filter across all 6 accounts
- [ ] **Multi-currency aware** - Shows USD and MXN correctly, with conversions
- [ ] **Transfer-aware** - Doesn't double-count BofA→Wise→Scotia
- [ ] **Fee-aware** - Tracks and displays Stripe fees, FX fees, wire fees
- [ ] **Tax-aware** - Can tag deductible (USA) vs deducible (Mexico) vs personal
- [ ] **Confidence-aware** - Shows when auto-categorization is uncertain
- [ ] **Provenance-aware** - All corrections logged with reason
- [ ] **Real-time + Historical** - Dashboard (now) + Reports (monthly/YoY)

---

## 🎓 Domain-Specific Vocabulary

**User's mental model:**

| User Says | System Concept | Notes |
|-----------|----------------|-------|
| "Cliente" | Counterparty (type: client) | Auto-detect from recurring deposits |
| "Contractor" | Counterparty (type: contractor) | Auto-detect from recurring Wise sends |
| "Transferencia" | Relationship (type: transfer) | BofA→Wise→Scotia chain |
| "Suscripción" | Series (type: subscription) | Fixed or variable recurring |
| "Deducible" | Tax categorization | Dual jurisdiction (USA + Mexico) |
| "Cuenta" | Account | BofA, Apple Card, Scotia, etc. |
| "Fee/Comisión" | Fee transaction | Linked to parent transaction |

---

## 🚀 Next Steps (How to Use This Document)

**For each vertical:**

1. **Before design:** Read relevant sections
   - Example: Vertical 2.1 (Transaction List) → Read "Reporting Requirements", "Transaction Examples"
   - Example: Vertical 3.5 (Relationships) → Read "Transfer Reconciliation Requirements"

2. **During design:** Validate against real patterns
   - Use transaction examples to test queries
   - Use error patterns to design edge case handling

3. **After design:** Cross-check primitives
   - Does primitive handle multi-currency? ✅/❌
   - Does primitive track confidence scores? ✅/❌
   - Does primitive log provenance? ✅/❌

---

**Last Updated:** 2025-05-23
**Source:** 41-question user interview
**Maintained by:** Eugenio Castro Garza
**Status:** Living document (update as new patterns discovered)
