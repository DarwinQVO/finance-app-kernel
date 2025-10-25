# ADR-0011: Multi-Jurisdiction Taxonomy Design

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Vertical 3.4 Tax Categorization

---

## Context

Support multiple tax jurisdictions (USA Schedule C, Mexico SAT, Canada T2125) with hierarchical categories and jurisdiction-specific deduction rules.

**The problem:**
- Tax categories are **jurisdiction-specific**:
  - USA IRS Schedule C has 27 expense lines (Line 9: Car & Truck, Line 25: Utilities)
  - Mexico SAT has different structure (CFDI categories: Gasoline, Professional Services)
  - Canada T2125 has overlapping but distinct categories
- Categories are **hierarchical**:
  - "Car & Truck" → "Gas" → "Premium Fuel"
  - "Professional Services" → "Legal" → "Contract Review"
- Deduction rates vary:
  - USA: 100% business use (full deduction), 50% personal/business (partial)
  - Mexico: Different rates per SAT category
- Users need to:
  - Map transactions to correct jurisdiction
  - Navigate hierarchical categories
  - Understand deduction implications

**Two approaches considered:**

1. **Flat global categories** — Simple dropdown, jurisdiction-agnostic
2. **Hierarchical per-jurisdiction** — Tree structure, jurisdiction-aware

---

## Decision

**Use hierarchical tree per jurisdiction with shared taxonomy engine.**

### Structure

```
Jurisdiction (USA, Mexico, Canada)
  └─ Category (Schedule C Line 9: Car & Truck)
      ├─ Subcategory (Gas)
      │   ├─ Premium Fuel
      │   └─ Diesel
      ├─ Subcategory (Maintenance)
      │   ├─ Oil Change
      │   └─ Tire Replacement
      └─ Metadata
          ├─ Deduction rate: 100% (business use)
          ├─ Form line: Schedule C Line 9
          └─ Documentation required: Mileage log
```

### Taxonomy Engine API

```python
class TaxonomyEngine:
    def get_categories(jurisdiction: str, parent_id: Optional[str] = None) -> List[Category]
    def get_deduction_rate(jurisdiction: str, category_id: str) -> float
    def validate_category_path(jurisdiction: str, category_id: str) -> bool
    def add_custom_category(jurisdiction: str, parent_id: str, name: str) -> Category
```

---

## Rationale

### 1. Compliance
- Each jurisdiction has **unique category structure**
  - Schedule C Line 9 ≠ SAT "Gasoline" (different scopes)
  - USA "Home Office" has specific deduction rules (safe harbor, actual expenses)
  - Mexico SAT requires CFDI-compliant categorization
- Hierarchical tree **matches official forms**
  - User sees: "Schedule C Line 9: Car & Truck > Gas"
  - Maps directly to IRS form line
- **Jurisdiction metadata** stores compliance rules
  - Deduction rates, documentation requirements, form mappings

### 2. Hierarchy Captures Relationships
- Real-world categories **naturally nest**:
  - "Professional Services" → "Legal" → "Contract Review"
  - More specific categories inherit parent rules
- **Drill-down UX** matches user mental model
  - Start broad ("Office Expenses"), refine ("Supplies > Paper")
- **Aggregation for reporting**
  - Sum all "Car & Truck" subcategories for Schedule C Line 9

### 3. Scalability
- Add new jurisdictions **without breaking existing**
  - Canada T2125 added without touching USA/Mexico code
  - Each jurisdiction has isolated tree
- **Shared taxonomy engine** avoids code duplication
  - Same tree traversal, validation, custom category logic
  - Jurisdiction-specific data in config files

### 4. Flexibility
- Users can **add custom categories** within jurisdiction
  - Example: Add "Software Subscriptions" under "Office Expenses"
  - Custom categories marked, not exported to tax forms (require accountant review)
- **Multi-level hierarchy** supports complex businesses
  - Freelancers: 2-3 levels deep
  - Small businesses: 4-5 levels (departments, projects, categories)

---

## Consequences

### ✅ Positive

- **Jurisdiction-compliant**: Categories match official tax forms (IRS, SAT, CRA)
- **Hierarchical navigation**: Intuitive drill-down UX (folder-like)
- **Scalable**: Add new jurisdictions (EU VAT, Australia ATO) without refactoring
- **Flexible custom categories**: Users add business-specific subcategories
- **Metadata-rich**: Deduction rates, form mappings, documentation requirements embedded

### ⚠️ Trade-offs

- **More complex UI**: Tree view instead of flat dropdown (more clicks to reach leaf category)
- **More validation rules**: Parent-child consistency, jurisdiction boundaries
- **Larger data model**: 500+ categories across 3 jurisdictions vs 100 flat categories
- **Custom category risk**: Users create non-compliant categories (mitigation: "Custom" flag, accountant review)

### 🔴 Risks (Mitigated)

- **Risk**: Jurisdiction rules change (IRS updates Schedule C)
  - **Mitigation**: Versioned taxonomy (USA_2024, USA_2025), migration scripts
  - **Mitigation**: Diff tool shows category changes year-over-year

- **Risk**: User confusion (which jurisdiction to use?)
  - **Mitigation**: Onboarding asks "Business location?" → pre-selects jurisdiction
  - **Mitigation**: Jurisdiction locked per account (not per-transaction)

- **Risk**: Complex migration (user switches jurisdiction mid-year)
  - **Mitigation**: Rare edge case (user relocates business)
  - **Mitigation**: Manual re-categorization tool + accountant review

---

## Alternatives Considered

### Alternative A: Flat Categories (Jurisdiction-Agnostic) — Rejected

**Structure:**
```
- Office Supplies
- Gas
- Legal Fees
- (No hierarchy, no jurisdiction)
```

**Pros:**
- Simpler UI (single dropdown)
- Fewer validation rules
- Faster selection (no drilling)

**Cons:**
- **No compliance support**: Categories don't map to tax forms
- **Ambiguity**: "Gas" for car or home heating? (context lost)
- **No jurisdiction-specific rules**: Can't encode deduction rates
- **Not scalable**: USA "Home Office" ≠ Mexico "Home Office" (different rules)

**Example failure:**
- User categorizes as "Gas"
- Tax report: Which Schedule C line? (Line 9: Car & Truck? Line 25: Utilities?)
- Accountant must manually re-map

### Alternative B: Single Global Taxonomy (Unified) — Rejected

**Structure:**
```
Global Category Tree
  ├─ Vehicle Expenses
  │   └─ Gas (maps to USA Schedule C Line 9, Mexico SAT "Gasoline")
  └─ Professional Services
      └─ Legal (maps to USA Schedule C Line 17, Mexico SAT "Legal Services")
```

**Pros:**
- Unified UI (one tree for all jurisdictions)
- Less duplication (one "Gas" category)

**Cons:**
- **Impossible to map accurately**: USA "Gas" scope ≠ Mexico "Gasoline" scope
- **Metadata explosion**: Each category needs 3+ jurisdiction-specific mappings
- **Conflicting rules**: USA allows home office deduction (safe harbor), Mexico doesn't
- **Breaks when jurisdictions diverge**: 2025 IRS changes don't affect Mexico

**Example failure:**
- Global "Legal Fees" category
- USA Schedule C Line 17 (100% deductible)
- Mexico SAT "Legal Services" (requires specific CFDI code)
- Canada T2125 Line 8862 (different documentation)
- One category can't satisfy all rules

### Alternative C: User-Defined Only (No Pre-built Taxonomy) — Rejected

**Structure:**
- Empty taxonomy, users create all categories

**Pros:**
- Maximum flexibility
- No jurisdiction assumptions

**Cons:**
- **No compliance support**: Users don't know tax form requirements
- **Training burden**: "What categories should I create?" (no guidance)
- **Inconsistent data**: User A calls it "Gas", User B calls it "Fuel"
- **Export difficulty**: Custom categories don't map to tax forms

**Reality check:**
- Target users: Solo entrepreneurs, small businesses (not accountants)
- Need pre-built, compliant categories out-of-the-box

---

## Implementation Notes

### Taxonomy Data Structure

```json
{
  "jurisdiction": "USA",
  "version": "2024",
  "categories": [
    {
      "id": "usa_schedule_c_line_9",
      "name": "Car & Truck Expenses",
      "form_reference": "Schedule C Line 9",
      "deduction_rate": 1.0,
      "documentation_required": "Mileage log or actual expense receipts",
      "children": [
        {
          "id": "usa_schedule_c_line_9_gas",
          "name": "Gas",
          "deduction_rate": 1.0,
          "children": [
            { "id": "usa_schedule_c_line_9_gas_premium", "name": "Premium Fuel" },
            { "id": "usa_schedule_c_line_9_gas_diesel", "name": "Diesel" }
          ]
        },
        {
          "id": "usa_schedule_c_line_9_maintenance",
          "name": "Maintenance",
          "children": [
            { "id": "usa_schedule_c_line_9_maintenance_oil", "name": "Oil Change" },
            { "id": "usa_schedule_c_line_9_maintenance_tire", "name": "Tire Replacement" }
          ]
        }
      ]
    }
  ]
}
```

### Custom Category Handling

```python
def add_custom_category(jurisdiction: str, parent_id: str, name: str) -> Category:
    # Validate parent exists and is in correct jurisdiction
    parent = taxonomy.get_category(jurisdiction, parent_id)
    if not parent:
        raise InvalidParentError()

    # Create custom category (marked as non-official)
    custom_cat = Category(
        id=generate_id(),
        name=name,
        parent_id=parent_id,
        custom=True,  # Not on official tax form
        deduction_rate=parent.deduction_rate,  # Inherit from parent
        jurisdiction=jurisdiction
    )

    taxonomy.save(custom_cat)
    return custom_cat
```

### UI Drill-Down Flow

1. User clicks "Categorize Transaction"
2. System pre-selects jurisdiction (USA)
3. Shows top-level categories:
   - Advertising
   - Car & Truck Expenses
   - Office Expenses
   - ...
4. User clicks "Car & Truck Expenses"
5. Shows subcategories:
   - Gas
   - Maintenance
   - Insurance
6. User clicks "Gas" → Transaction categorized as `usa_schedule_c_line_9_gas`

---

## Related Decisions

- **ADR-0010**: Auto-linked series transactions inherit series category (pre-filled)
- **ADR-0012**: Transfer pairs not categorized (intra-account moves exempt from tax)
- **Future**: ADR-0021 will add AI-powered category suggestion (based on merchant name)

---

**References:**
- Vertical 3.4: [docs/verticals/3.4-tax-categorization.md](../verticals/3.4-tax-categorization.md)
- TaxonomyEngine: [docs/primitives/ol/TaxonomyEngine.md](../primitives/ol/TaxonomyEngine.md)
- Schema: [docs/schemas/tax-category.schema.json](../schemas/tax-category.schema.json)
