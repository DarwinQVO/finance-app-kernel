# ADR-0046: Narrative-First Documentation (Storytelling)

**Status**: Accepted
**Date**: 2025-10-27
**Context**: Phase 1 (Storytelling docs), Phase 3 (Finance App Implementation mapping)
**Deciders**: System architects, product team
**Related**: ADR-0042 (Simplicity Profiles), All 33 primitive docs

---

## Context

When documenting a universal kernel (87 OL primitives, 56 IL components, 23 verticals), we face a fundamental question:

**How do we make abstract machinery understandable?**

**Traditional approaches:**
1. **API reference**: Alphabetical list of classes/methods
2. **Architecture diagrams**: Box-and-arrow system overview
3. **Code comments**: Inline documentation

**The Problem:**
- API reference lacks context ("What problem does this solve?")
- Architecture diagrams are static ("How does data flow?")
- Code comments are scattered ("What's the big picture?")

**The Question:**
How do we document complex abstractions in a way that's **concrete, contextual, and compelling**?

---

## Decision

**We use Narrative-First Documentation (Storytelling):**

1. **Start with user journeys** (not API reference)
2. **Tell stories with real users** (not abstract "users")
3. **Show data flow through system** (not static architecture)
4. **Document in user's language** (Spanish for this project)
5. **Code examples are executable** (not pseudocode)

**Structure:**
```
Phase 1: Storytelling docs (narrative foundation)
  ├─ 1-core-user-journeys.md    (12 journeys: what user does)
  ├─ 2-data-model.md             (data structures: what gets stored)
  ├─ 3-ui-flows.md               (wireframes: what user sees)
  ├─ 4-parsing-logic.md          (extraction: how PDFs become data)
  └─ 5-open-questions.md         (decisions: what we're unsure about)

Phase 2: Simplicity Profiles (show scaling)
  └─ 33 primitives × 3 profiles (Personal → Small Business → Enterprise)

Phase 3: Implementation mapping (prove it works)
  └─ finance-app-implementation.md (Darwin uses 3.8% of Enterprise code)
```

**Key principle:** **Abstractions explained through concrete examples, not the reverse.**

---

## Rationale

### 1. Narratives Provide Context

**Traditional API doc:**
```markdown
## HashCalculator

**Method:** `calculate_hash(data: bytes) -> str`

Calculates SHA-256 hash of input data.

**Parameters:**
- data: Bytes to hash

**Returns:**
- String hash with algorithm prefix (e.g., "sha256:abc123...")
```

**Problem:**
- No context: Why hash? When to use?
- No motivation: What problem does this solve?
- No alternatives: Why SHA-256 not MD5?

**Narrative doc (Phase 1):**
```markdown
## Journey 1: When User Uploads Document

El usuario abre la aplicación de finanzas un domingo por la mañana,
hace clic en "Subir Estado de Cuenta", selecciona `BoFA_Oct2024.pdf`
(2.3MB), lo sube. El sistema acepta inmediatamente sin preguntas.

**Problem:** Prevent duplicate uploads (user uploads same file twice)

**Solution:** Calculate file hash for deduplication
```python
hash_calculator = HashCalculator(algorithm="sha256")
file_hash = hash_calculator.calculate_hash(pdf_bytes)
# Returns: "sha256:abc123..."

# Check if already stored
if storage_engine.exists(file_hash):
    return "File already uploaded (duplicate detected)"
```

**Why SHA-256:**
- Collision-resistant (probability of duplicate hash ≈ 0)
- Industry standard for content-addressable storage
- Faster than SHA-512, more secure than MD5

**Personal Profile:** In-memory hashing (files <10MB)
**Enterprise Profile:** Streaming hashing (files up to 10GB)
```

**Benefits:**
- Context: User uploads → need deduplication
- Motivation: Prevent duplicates (real problem)
- Alternatives: SHA-256 justified vs MD5/SHA-512
- Scaling: Personal vs Enterprise implementations shown

---

### 2. Real Users Beat Abstract "Users"

**Traditional doc:**
```markdown
The user uploads a document and views transactions.
```

**Problem:**
- Who is "the user"? (CEO? Accountant? Developer?)
- What documents? (PDFs? CSVs? Images?)
- How many transactions? (10? 1000? 1M?)

**Narrative doc (Phase 1):**
```markdown
## El Usuario

**Contexto:**
- Nombre: "El usuario" (generic for privacy)
- Uso: Personal (500 transacciones/mes, 1 usuario)
- Cuentas: 2 (BoFA Checking, Apple Card)
- Frecuencia: Sube estados de cuenta 1× al mes (12× al año)

## Journey 1: Upload domingo por la mañana

El usuario abre la aplicación el domingo 27 de octubre a las 10:30 AM.
Tiene su estado de cuenta de BoFA en Descargas: `BoFA_Oct2024.pdf` (2.3MB, 42 transacciones).

Arrastra el PDF al cuadro de subida. La app responde:
- ✓ Archivo aceptado (2.3 MB, PDF)
- ⏳ Procesando... ~30 segundos
- ✓ 42 transacciones extraídas
- → Redirige a vista de transacciones
```

**Benefits:**
- Concrete: Real file size (2.3MB), transaction count (42), timestamp (10:30 AM)
- Relatable: Sunday morning routine (universal context)
- Scale-aware: 500 tx/month clarifies "personal use"
- Language: Spanish (user's language)

---

### 3. Literate Programming Style

**Traditional doc:**
- Code → Comments → API reference
- Bottom-up (implementation details first)

**Literate programming:**
- Story → Problem → Solution → Code
- Top-down (user journey first)

**Example from `4-parsing-logic.md`:**

```markdown
## Casos Especiales

### 7. Transacciones en Moneda Extranjera

**Cómo se ven:**
```
10/15/2024  RESTAURANT PARIS EUR 45.00
            EXCHANGE RATE 1.10                -$49.50
10/15/2024  FOREIGN TRANSACTION FEE            -$2.50
```

**Manejo:**
- Dos transacciones separadas:
  1. Cargo principal (EUR 45.00 convertido a $49.50)
  2. Comisión FX ($2.50)
- Parsear ambas, vincular más tarde en detección de relaciones

**Código:**
```python
def extract_fx_info(desc_raw):
    """Extraer información de moneda extranjera de la descripción"""
    currency_match = re.search(r'([A-Z]{3})\s+([\d.]+)', desc_raw)
    rate_match = re.search(r'EXCHANGE RATE\s+([\d.]+)', desc_raw)

    return {
        'foreign_currency': currency_match.group(1) if currency_match else None,
        'foreign_amount': float(currency_match.group(2)) if currency_match else None,
        'exchange_rate': float(rate_match.group(1)) if rate_match else None
    }
```
```

**Structure:**
1. **Show the data** (real PDF excerpt)
2. **Explain the problem** (2 transactions, how to handle?)
3. **Document the solution** (parse both, link later)
4. **Provide executable code** (real Python, not pseudocode)

**Benefits:**
- Reader sees REAL data first (concrete before abstract)
- Problem motivated by example (not "imagine if...")
- Code is executable (can copy-paste and run)
- Spanish narrative, English code (bilingual technical context)

---

### 4. Documentation as Design Tool

**Traditional approach:**
- Write code first
- Document later ("we'll add docs before shipping")
- Documentation often incomplete or outdated

**Narrative-first approach:**
- Write stories first (Phase 1)
- Stories reveal missing abstractions
- Code implements stories

**Example from Phase 1 → Phase 2:**

**Phase 1 story identified need:**
```markdown
## Journey 2: When System Parses Document

Sistema comienza a parsear `BoFA_Oct2024.pdf`:
- Extrae 42 observaciones crudas (AS-IS del PDF)
- Normaliza fechas: "10/15/2024" → "2024-10-15"
- Limpia montos: "$87.43" → -87.43
- Categoriza comerciantes: "WHOLE FOODS" → "Groceries"
```

**Phase 2 abstraction emerged:**
```python
# ObservationStore primitive (needed to store raw extractions)
# NormalizationLog primitive (needed to track transformations)
# MerchantNormalizer primitive (needed to clean merchant names)
# CategoryRules primitive (needed to assign categories)
```

**Story drove design:**
- Concrete user journey revealed 4 needed primitives
- Without story, might have built 1 monolithic "Parser" (wrong abstraction level)

---

### 5. Translation to User's Language

**Traditional tech docs:**
- English only (excludes non-English speakers)
- Technical jargon (excludes non-technical stakeholders)

**Our approach:**
- **Narrative in Spanish** (user's language)
- **Code in English** (universal technical language)
- **Bilingual glossary** (bridge between languages)

**Example from `1-core-user-journeys.md`:**

```markdown
## Journey 1: Cuando el Usuario Sube Documento

El usuario abre la aplicación de finanzas un domingo por la mañana...

[Spanish narrative continues...]

**Primitivos del Kernel Usados:**
- `HashCalculator.calculate_hash()` - Calcular hash del archivo
- `StorageEngine.store()` - Almacenar PDF
- `FileArtifact.create()` - Crear registro de metadatos
```

**Benefits:**
- Spanish speakers understand user journey (accessibility)
- English code maintains technical universality
- Non-developers understand WHAT system does
- Developers understand HOW system works

---

## Consequences

### Positive

✅ **Concrete examples**: Abstractions explained via real use cases (Darwin's finance app)
✅ **User-centric**: Documentation starts with user journey, not API reference
✅ **Discoverable**: Stories answer "What problem does X solve?" before "How do I call X?"
✅ **Executable code**: Examples are real Python/SQL, not pseudocode
✅ **Bilingual**: Spanish narrative, English code (accessible + technical)
✅ **Design tool**: Stories reveal missing abstractions, drive architecture
✅ **Maintainable**: Outdated stories break tests (unlike orphaned API docs)

### Negative

⚠️ **Upfront cost**: Stories take longer to write than API reference
⚠️ **Translation overhead**: Maintaining Spanish + English adds complexity
⚠️ **Story-code drift**: Stories can become outdated if code changes

### Mitigations

**Upfront cost:**
- Paid back in clarity (fewer "How do I..." questions)
- Stories prevent over-engineering (YAGNI applied early)

**Translation overhead:**
- Acceptable for projects with non-English stakeholders
- Narrative in user's language, code universal

**Story-code drift:**
- Tests validate stories (executable examples)
- Phase 3 (Implementation mapping) ties stories to code

---

## Documentation Structure

### Phase 1: Storytelling Foundation

**5 narrative documents:**

1. **1-core-user-journeys.md** (618 líneas)
   - 12 user journeys from upload to tax categorization
   - Real user context (500 tx/month, 2 accounts)
   - Data flow: Upload → Parse → View → Normalize → Corrections

2. **2-data-model.md** (374 líneas)
   - SQL schemas with inline comments
   - Bitemporal model explained via examples
   - Canonical ID strategy (UUID v5)

3. **3-ui-flows.md** (640 líneas)
   - ASCII wireframes (desktop + mobile)
   - User actions per screen
   - Loading states, empty states, error states

4. **4-parsing-logic.md** (618 líneas)
   - Real PDF structure (BoFA statement example)
   - 10 edge cases (pending transactions, refunds, FX fees)
   - Executable Python parsers

5. **5-open-questions.md** (556 líneas)
   - 10 undecided features (multi-currency, budgets, mobile app)
   - Decision criteria per feature
   - YAGNI philosophy explained

**Total:** 2,806 lines of narrative (before any code written)

---

### Phase 2: Simplicity Profiles

**33 primitives × 3 profiles each:**

Each primitive documented with:
- **Personal Profile** (0-100 LOC): Darwin's implementation
- **Small Business Profile** (50-300 LOC): 10-user company
- **Enterprise Profile** (200-1000 LOC): 1000-user platform

**Example:** `StorageEngine.md`
```markdown
## Personal Profile (30 LOC)
**Use when:** <1K files, local machine
**Implementation:** Filesystem (~/. finance-app/storage/)
**Features:** Content-addressable storage, deduplication
**NOT included:** S3, encryption, compression, GC

## Small Business Profile (150 LOC)
**Use when:** <100K files, single server
**Implementation:** Filesystem + PostgreSQL metadata
**Features:** + Metadata queries, basic monitoring
**NOT included:** S3, distributed storage

## Enterprise Profile (800 LOC)
**Use when:** 100K+ files, distributed
**Implementation:** S3 + CloudFront + PostgreSQL
**Features:** + Encryption, compression, GC, metrics, quotas
```

---

### Phase 3: Implementation Mapping

**1 comprehensive mapping document:**

`docs/kernel/examples/finance-app-implementation.md` (1,402 líneas)

- Maps 12 user journeys to 33 primitives
- Shows Darwin's configuration (Personal Profile)
- Proves kernel works at 500 tx/month scale
- Documents features used (8/12) vs skipped (4/12)

**Key finding:** Darwin uses 1,365 LOC (3.8% of Enterprise's 35,700 LOC)

---

## Alternatives Considered

### Alternative 1: API Reference Only

**Approach:** Alphabetical list of classes/methods (Javadoc style).

```markdown
## HashCalculator

### Methods

#### calculate_hash(data: bytes) -> str
Calculates hash of data.

**Parameters:**
- data: Input bytes

**Returns:**
- Hash string
```

**Rejected because:**
- No context (Why hash? When to use?)
- No motivation (What problem solved?)
- No examples (How to call?)
- No scaling (Personal vs Enterprise?)

---

### Alternative 2: Architecture Diagrams Only

**Approach:** Box-and-arrow system overview.

```
[User] → [Upload API] → [Parser] → [Storage] → [Database]
```

**Rejected because:**
- Static (doesn't show data flow)
- Abstract (no real examples)
- No code (how to implement?)

---

### Alternative 3: Code-First (Comments Only)

**Approach:** Document via code comments.

```python
class HashCalculator:
    """Calculates hashes of files."""

    def calculate_hash(self, data: bytes) -> str:
        """Calculate SHA-256 hash."""
        return f"sha256:{hashlib.sha256(data).hexdigest()}"
```

**Rejected because:**
- Scattered (no big picture)
- Implementation-focused (not user-focused)
- No user journey (when to use?)

---

## Implementation Notes

### Narrative Template

```markdown
## Journey {N}: {Title}

**Contexto del Usuario:**
- Escenario: {What user is doing}
- Datos: {What data involved}
- Escala: {Transaction count, file size, frequency}

**Flujo de Datos:**
1. Usuario hace X
2. Sistema procesa Y
3. Usuario ve Z

**Primitivos del Kernel Usados:**
- `Primitive1.method()` - Propósito
- `Primitive2.method()` - Propósito

**Configuración (Personal Profile):**
```yaml
primitive1:
  enabled: true
  config: ...
```

**Código Ejecutable:**
```python
# Real implementation (copy-paste and run)
result = primitive1.method(input_data)
```

**Casos Especiales:**
- Edge case 1: How handled
- Edge case 2: How handled
```

---

## Related Decisions

- **ADR-0042:** Simplicity Profiles Pattern (narrative shows 3 scaling tiers)
- **ADR-0043:** Three-Tier Scaling Strategy (stories explain Personal vs Enterprise)
- **ADR-0044:** YAGNI Principle (stories document skipped features explicitly)
- **ADR-0045:** Hardcoded Configuration (stories show when dicts beat databases)

---

## Review & Updates

- **2025-10-27:** Initial decision (Phase 1-3 completion proves approach effective)
- Next review: After 10+ developers use documentation (measure comprehension vs API reference)

---

**Decision**: Use Narrative-First Documentation. Start with user journeys (Phase 1), add scaling profiles (Phase 2), prove with real implementation (Phase 3).
**Status**: Accepted and implemented across all project documentation (2,806 lines storytelling, 33 primitives profiled, 1,402 lines implementation mapping).
