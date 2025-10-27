# Modelo de Datos (Mínimo)

> **Propósito:** Definir las estructuras de datos principales necesarias para la aplicación de finanzas (simple, práctica, sin complejidad empresarial)

---

## Filosofía de Diseño

**Empezar Simple, Crecer Después**

Este modelo de datos representa lo que necesitamos AHORA para 500 transacciones/mes, no lo que podríamos necesitar para 10M transacciones/mes.

Decisiones clave:
- ❌ **Sin seguimiento bitemporal** (transaction_time/valid_time) - se puede agregar después si es necesario
- ❌ **Sin ledger de proveniencia** (audit trail completo) - un simple audit_log es suficiente
- ❌ **Sin claves de sharding** (partition_id, shard_id) - base de datos SQLite única
- ✅ **Claves foráneas simples** (account_id, category_id, counterparty_id)
- ✅ **Tipos directos** (TEXT, REAL, INTEGER, DATETIME)

---

## Entidades Principales

### 1. Transaction (8 campos)

**Propósito:** Representa una sola transacción financiera

```sql
CREATE TABLE transactions (
  id           TEXT PRIMARY KEY,      -- TX_abc123
  date         DATE NOT NULL,         -- 2024-10-15
  amount       REAL NOT NULL,         -- -87.43 (negativo = gasto)
  merchant     TEXT NOT NULL,         -- "Whole Foods Market"
  category_id  TEXT,                  -- FK to categories.id
  account_id   TEXT NOT NULL,         -- FK to accounts.id
  notes        TEXT,                  -- Notas agregadas por el usuario
  tags         TEXT,                  -- Separado por comas: "groceries,weekly"

  created_at   DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Decisiones de Campos:**

- **id**: UUID generado con prefijo TX_ (legible en logs)
- **date**: Solo DATE (sin hora) - extractos bancarios no incluyen hora
- **amount**: Decimal con signo (negativo = gasto, positivo = ingreso)
- **merchant**: Nombre normalizado del comerciante (no descripción cruda del PDF)
- **category_id**: FK a categories (NULL = sin categorizar)
- **account_id**: Requerido - toda transacción pertenece a una cuenta
- **notes**: Usuario puede agregar contexto (ej., "dividido con roommate")
- **tags**: Sistema flexible de etiquetas (ej., "deducible-impuestos", "reembolsable")

**Lo que NO está en esta tabla:**
- ❌ Datos crudos del PDF (almacenados separadamente en observations)
- ❌ Historial de normalización (almacenado separadamente en audit_log)
- ❌ Código de moneda (asumido USD para v1)
- ❌ Tasa de cambio (se puede agregar después para multi-moneda)

---

### 2. Category (3 campos)

**Propósito:** Organizar transacciones en categorías de gasto

```sql
CREATE TABLE categories (
  id          TEXT PRIMARY KEY,      -- CAT_groceries
  name        TEXT NOT NULL UNIQUE,  -- "Groceries"
  parent_id   TEXT,                  -- FK to categories.id (para subcategorías)
  color       TEXT,                  -- "#4CAF50" (color hex para UI)

  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Estructura Jerárquica:**

```
Groceries (CAT_groceries)
  ├─ Supermarkets (CAT_groceries_supermarkets)
  ├─ Farmers Markets (CAT_groceries_farmers)
  └─ Specialty Stores (CAT_groceries_specialty)

Food & Dining (CAT_food)
  ├─ Restaurants (CAT_food_restaurants)
  ├─ Coffee Shops (CAT_food_coffee)
  └─ Fast Food (CAT_food_fastfood)
```

**Categorías Iniciales (20-30 categorías comunes):**
```
- Groceries
- Rent/Mortgage
- Utilities (Electric, Water, Gas)
- Transportation (Gas, Parking, Uber)
- Food & Dining
- Entertainment (Movies, Streaming, Concerts)
- Healthcare (Doctor, Pharmacy, Insurance)
- Shopping (Clothing, Electronics, Home)
- Travel (Flights, Hotels, Vacation)
- Personal Care (Haircut, Gym)
- Insurance (Car, Home, Life)
- Taxes (Federal, State, Property)
- Charitable Donations
- Savings & Investments
- Other
```

**Profundidad máxima:** 3 niveles (Categoría → Subcategoría → Sub-subcategoría)

---

### 3. Account (4 campos)

**Propósito:** Cuentas financieras del usuario (checking, savings, tarjetas de crédito)

```sql
CREATE TABLE accounts (
  id           TEXT PRIMARY KEY,      -- ACC_chase_checking
  name         TEXT NOT NULL,         -- "Chase Checking"
  institution  TEXT NOT NULL,         -- "Chase Bank"
  last4        TEXT NOT NULL,         -- "1234" (últimos 4 dígitos)
  type         TEXT NOT NULL,         -- checking, savings, credit_card

  active       BOOLEAN DEFAULT TRUE,  -- Soft delete
  created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Tipos de Cuenta:**
- `checking` - Cuenta bancaria corriente
- `savings` - Cuenta de ahorros
- `credit_card` - Tarjeta de crédito
- `investment` - Cuenta de inversión/corretaje (futuro)
- `loan` - Préstamo/hipoteca (futuro)

**¿Por qué last4?**
- Extractos bancarios muestran "Account: ****1234"
- Ayuda al usuario a identificar qué cuenta sin mostrar número completo
- Usado para resolución de cuenta al importar extractos

**Bandera active:**
- Soft delete - nunca eliminar físicamente cuentas (transacciones históricas las referencian)
- Usuario puede "archivar" cuentas antiguas (active=false)
- Cuentas archivadas ocultas en UI pero preservadas en base de datos

---

## Entidades de Soporte

### 4. Counterparty (simplificado)

**Propósito:** Comerciantes, empleadores, otras entidades con las que el usuario hace transacciones

```sql
CREATE TABLE counterparties (
  id            TEXT PRIMARY KEY,      -- CP_wholefoods
  canonical_name TEXT NOT NULL UNIQUE, -- "Whole Foods Market"
  type          TEXT NOT NULL,         -- merchant, employer, person, other

  created_at    DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE counterparty_aliases (
  counterparty_id TEXT NOT NULL,       -- FK to counterparties.id
  alias           TEXT NOT NULL,       -- "WHOLE FOODS MKT #1234"

  PRIMARY KEY (counterparty_id, alias)
);
```

**¿Por qué tabla separada de aliases?**
- Un comerciante puede tener muchas variaciones en extractos bancarios
- "WHOLE FOODS MARKET #1234 SF", "WFM", "WHOLE FOODS MKT" → todos misma contraparte
- Permite coincidencia difusa sin duplicar datos

---

### 5. Series (pagos recurrentes)

**Propósito:** Rastrear transacciones recurrentes (suscripciones, renta, etc.)

```sql
CREATE TABLE series (
  id          TEXT PRIMARY KEY,      -- SER_netflix
  name        TEXT NOT NULL,         -- "Netflix Subscription"
  amount      REAL NOT NULL,         -- 14.99
  tolerance   REAL DEFAULT 1.0,      -- ± $1 varianza permitida
  frequency   TEXT NOT NULL,         -- monthly, weekly, yearly
  day_of_month INTEGER,              -- 15 (para mensual)
  counterparty_id TEXT,              -- FK to counterparties.id
  category_id TEXT,                  -- FK to categories.id

  active      BOOLEAN DEFAULT TRUE,
  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE series_instances (
  id              TEXT PRIMARY KEY,  -- SI_netflix_oct2024
  series_id       TEXT NOT NULL,     -- FK to series.id
  transaction_id  TEXT NOT NULL,     -- FK to transactions.id
  expected_date   DATE NOT NULL,     -- 2024-10-15
  actual_date     DATE,              -- 2024-10-16 (1 día tarde)
  expected_amount REAL NOT NULL,     -- 14.99
  actual_amount   REAL,              -- 15.99 (aumento de precio)
  variance        REAL,              -- 1.00
  status          TEXT NOT NULL,     -- matched, variance, missing

  created_at      DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Valores de status:**
- `matched` - Transacción encontrada, monto dentro de tolerancia
- `variance` - Transacción encontrada, monto fuera de tolerancia (marcar para revisión)
- `missing` - Transacción esperada no encontrada (cancelada o pendiente)

---

### 6. Reglas de Categorización (config YAML, no base de datos)

**Propósito:** Definir cómo auto-categorizar transacciones

**Archivo: `config/categorization_rules.yaml`**

```yaml
rules:
  - pattern: "WHOLE FOODS"
    category: "Groceries"
    confidence: 0.9

  - pattern: "NETFLIX"
    category: "Entertainment"
    confidence: 0.95

  - pattern: "SHELL|CHEVRON|76"
    category: "Transportation"
    confidence: 0.85

  - pattern: "STARBUCKS|PEETS|PHILZ"
    category: "Food & Dining"
    subcategory: "Coffee Shops"
    confidence: 0.9

  - pattern: "UBER|LYFT"
    category: "Transportation"
    subcategory: "Rideshare"
    confidence: 0.95
```

**¿Por qué YAML en lugar de base de datos?**
- Más fácil de versionar (git)
- Más fácil de editar en masa (editor de texto)
- Más fácil de compartir/importar/exportar
- Se puede cargar en memoria al inicio (50-100 reglas = <1MB)

**Se puede mover a base de datos después si:**
- Usuario necesita UI web para editar reglas
- Múltiples usuarios necesitan conjuntos de reglas separados
- Reglas exceden 1000+ (consideración de rendimiento)

---

## Auditoría y Metadatos

### 7. Audit Log (simple)

**Propósito:** Rastrear acciones del usuario para accountability (no proveniencia completa)

```sql
CREATE TABLE audit_log (
  id           TEXT PRIMARY KEY,      -- LOG_abc123
  action       TEXT NOT NULL,         -- "category_changed", "transaction_deleted"
  entity_type  TEXT NOT NULL,         -- "transaction", "category", "account"
  entity_id    TEXT NOT NULL,         -- TX_abc123, CAT_groceries
  old_value    TEXT,                  -- JSON: {"category_id": "CAT_shopping"}
  new_value    TEXT,                  -- JSON: {"category_id": "CAT_books"}
  user_id      TEXT,                  -- "user" (para futuro multi-usuario)
  timestamp    DATETIME DEFAULT CURRENT_TIMESTAMP,

  INDEX idx_entity (entity_type, entity_id),
  INDEX idx_timestamp (timestamp)
);
```

**Lo que se registra:**
- Cambios de categoría
- Eliminaciones de transacción
- Ediciones de cuenta
- Correcciones manuales

**Lo que NO se registra (todavía):**
- Cada consulta a base de datos (demasiado verboso)
- Cada vista de página (no necesario para v1)
- Operaciones del sistema (parsing, normalización) - logs separados para esto

---

### 8. Upload Records (máquina de estados)

**Propósito:** Rastrear cargas de PDF y estado de procesamiento

```sql
CREATE TABLE upload_records (
  id              TEXT PRIMARY KEY,      -- UL_abc123
  filename        TEXT NOT NULL,         -- "bofa-statement-october.pdf"
  file_size       INTEGER NOT NULL,      -- 1245678 (bytes)
  file_hash       TEXT NOT NULL UNIQUE,  -- Hash SHA-256 (para deduplicación)
  status          TEXT NOT NULL,         -- queued, parsing, parsed, error
  uploaded_by     TEXT,                  -- "user"
  uploaded_at     DATETIME DEFAULT CURRENT_TIMESTAMP,
  processed_at    DATETIME,
  error_message   TEXT,

  observations_count INTEGER DEFAULT 0,  -- Cuántas transacciones crudas se extrajeron

  INDEX idx_status (status),
  INDEX idx_hash (file_hash)
);
```

**Máquina de estados de status:**
```
queued → parsing → parsed → (completo)
          ↓
        error
```

**Deduplicación:**
- Si file_hash existe → retornar 409 Conflict antes de parsear
- Previene importaciones duplicadas

---

## Diagrama de Relaciones

```
┌─────────────┐
│  uploads    │
└─────────────┘
       │
       ↓ (crea)
┌─────────────┐
│observations │ (datos crudos del PDF - tabla separada)
└─────────────┘
       │
       ↓ (normaliza a)
┌─────────────┐        ┌─────────────┐
│transactions │───────→│  accounts   │
└─────────────┘        └─────────────┘
       │                      ↑
       ↓                      │
┌─────────────┐               │
│ categories  │               │
└─────────────┘               │
       ↑                      │
       │                      │
┌─────────────┐        ┌─────────────┐
│counterparties│       │   series    │
└─────────────┘        └─────────────┘
       │                      │
       ↓                      ↓
┌─────────────┐        ┌─────────────┐
│   aliases   │        │  instances  │
└─────────────┘        └─────────────┘
```

---

## Lo que Deliberadamente se Excluye (Se Puede Agregar Después)

**Seguimiento Bitemporal:**
- Sin columnas `transaction_time` / `valid_time`
- Timestamp simple `updated_at` es suficiente para v1
- Se puede agregar después si auditorías fiscales requieren "¿qué sabía en fecha X?"

**Multi-Moneda:**
- Sin campo `currency_code`
- Sin campo `exchange_rate`
- Asumido USD para v1
- Se puede agregar después si usuario viaja internacionalmente con frecuencia

**Ledger de Proveniencia:**
- Sin event store de solo-agregar
- Simple audit_log con capacidad UPDATE es suficiente
- Se puede agregar después si compliance requiere audit trail inmutable

**Sistema de Etiquetas:**
- Campo simple `tags` separado por comas
- Sin tabla separada `tags` con relación muchos-a-muchos
- Se puede normalizar después si gestión de etiquetas se vuelve compleja

**Adjuntos:**
- Sin tabla `attachments` (recibos, facturas)
- Usuario puede enlazar notas como "recibo en Dropbox: /tax-docs/2024/receipt-123.pdf"
- Se puede agregar después si gestión de recibos se vuelve importante

**Seguimiento de Presupuesto:**
- Sin tabla `budgets`
- Se puede calcular gasto por categoría manualmente
- Se puede agregar después si usuario quiere alertas ("gastado 90% de presupuesto de groceries")

---

## Índices de Base de Datos

**Críticos para Rendimiento:**

```sql
-- Transactions
CREATE INDEX idx_transactions_date ON transactions(date);
CREATE INDEX idx_transactions_account ON transactions(account_id);
CREATE INDEX idx_transactions_category ON transactions(category_id);
CREATE INDEX idx_transactions_date_account ON transactions(date, account_id);

-- Audit Log
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_timestamp ON audit_log(timestamp DESC);

-- Upload Records
CREATE INDEX idx_uploads_status ON upload_records(status);
CREATE INDEX idx_uploads_hash ON upload_records(file_hash);
```

**¿Por qué estos índices?**
- `date` - La mayoría de consultas filtran por rango de fechas
- `account_id` - "Mostrar transacciones para Chase Checking"
- `category_id` - "Mostrar todo gasto en Groceries"
- `date + account_id` - Índice compuesto para consulta combo común
- `status` - Worker consulta para `WHERE status = 'queued'`

---

## Tamaños de Datos (500 tx/mes)

**Estimaciones de Almacenamiento:**

| Tabla | Filas/Año | Tamaño/Fila | Total/Año |
|-------|-----------|----------|------------|
| transactions | 6,000 | ~200 bytes | 1.2 MB |
| categories | 30 | ~100 bytes | 3 KB |
| accounts | 6 | ~100 bytes | 600 bytes |
| counterparties | 200 | ~150 bytes | 30 KB |
| series | 10 | ~200 bytes | 2 KB |
| upload_records | 120 | ~300 bytes | 36 KB |
| audit_log | 500 | ~400 bytes | 200 KB |

**Tamaño Total de Base de Datos:** ~1.5 MB/año

**Con 5 años de datos:** ~7.5 MB (SQLite maneja esto fácilmente)

**PDFs:** 120 PDFs/año × 2 MB = 240 MB/año × 5 años = 1.2 GB

**Almacenamiento Total:** ~1.2 GB para 5 años de datos (cabe en cualquier laptop)

---

## Estrategia de Migración

**Cuándo actualizar a PostgreSQL:**
- Transacciones exceden 50,000 (tomaría 8+ años a ritmo actual)
- Múltiples usuarios concurrentes
- Necesidad de seguridad a nivel de fila
- Distribución geográfica (réplicas de lectura)

**Cuándo agregar seguimiento bitemporal:**
- Auditoría fiscal requiere consultas "¿qué sabía en fecha X?"
- Compliance regulatorio (GDPR, SOX)
- Necesidad de reconstruir estados históricos

**Cuándo separar tabla observations:**
- Necesidad de re-procesar datos históricos con nuevos parsers
- Querer hacer A/B testing de diferentes reglas de normalización

---

## Resumen

**Este modelo de datos es:**
- ✅ Simple (8 tablas, ~50 columnas total)
- ✅ Suficiente para 500 tx/mes
- ✅ Compatible con SQLite
- ✅ Extensible (se pueden agregar campos después)

**Este modelo de datos NO es:**
- ❌ Escala empresarial (sin sharding, sin particionamiento)
- ❌ Multi-tenant (sin aislamiento tenant_id)
- ❌ Listo para compliance (sin audit trail inmutable)

**Insight Clave:**
Empezar con schema simple que resuelve necesidades reales. Agregar complejidad solo cuando puntos de dolor emergen, no en anticipación de problemas teóricos.
