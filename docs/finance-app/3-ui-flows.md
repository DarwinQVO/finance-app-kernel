# Flujos de Interfaz de Usuario

> **Propósito:** Definir pantallas de interfaz de usuario e interacciones (wireframes + acciones del usuario)

---

## Pantalla 1: Subir Archivo

**Filosofía de Un Botón:** El usuario no debería necesitar un manual

**Wireframe para Escritorio:**

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

**Wireframe para Móvil:**

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

**Acciones del Usuario:**

1. **Soltar archivo o hacer clic en Explorar**
   - Se abre el selector de archivos (escritorio: nativo, móvil: cámara/archivos)
   - El usuario selecciona `bofa-statement-october.pdf`

2. **Comienza la subida**
   ```
   ┌──────────────────────────────────────────┐
   │ Subiendo bofa-statement-october.pdf      │
   │ ████████████░░░░░░░░░░░░░░░ 45%  1.2 MB │
   └──────────────────────────────────────────┘
   ```

3. **Estado de procesamiento**
   ```
   ┌──────────────────────────────────────────┐
   │ ✓ Subida completa                        │
   │ ⏳ Procesando estado de cuenta... ~30 seg│
   │                                          │
   │ [Ver Dashboard]  [Subir Otro]           │
   └──────────────────────────────────────────┘
   ```

4. **Estados de error**
   ```
   ❌ Subida Fallida: El archivo no es un PDF
   ❌ Error de Procesamiento: No se pudo leer el PDF - el archivo puede estar corrupto
   ⚠️  Duplicado: Este archivo fue subido el 15 de octubre
   ```

---

## Pantalla 2: Lista de Transacciones (Pantalla Principal)

**Tabla con Filtros:** Diseño clásico de aplicación financiera

**Wireframe para Escritorio:**

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

**Wireframe para Móvil:**

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

**Acciones del Usuario:**

1. **Filtrar por fecha**
   - Clic en "[Últimos 30 días ▼]" → Se abre el menú desplegable
   - Opciones: Este Mes, Mes Pasado, Últimos 3 Meses, Rango Personalizado, Todo el Tiempo
   - Seleccionar "Este Mes" → La tabla se actualiza

2. **Filtrar por cuenta**
   - Clic en "[Todas las Cuentas ▼]" → Se abre el menú desplegable
   - Opciones: Todas las Cuentas, Chase Checking, Chase Savings, BoFA Credit Card, etc.
   - Seleccionar "Chase Checking" → La tabla muestra solo transacciones de cuenta corriente

3. **Buscar**
   - Escribir "starbucks" en el cuadro de búsqueda
   - La tabla filtra para mostrar solo transacciones de Starbucks
   - Limpiar búsqueda → Muestra todas las transacciones nuevamente

4. **Hacer clic en transacción**
   - Se abre un modal mostrando detalles de la transacción (ver Pantalla 3)

5. **Ordenar columnas**
   - Clic en encabezado de columna "Date" → Ordenar por fecha descendente
   - Clic en encabezado de columna "Amount" → Ordenar por monto descendente
   - El indicador de flecha muestra la dirección de ordenamiento ▼ ▲

6. **Paginación**
   - Clic en "Next >" → Muestra transacciones 51-100
   - Clic en número de página "2" → Salta a la página 2
   - Paginación simple por offset (no basada en cursor por simplicidad)

---

## Pantalla 3: Modal de Detalle de Transacción

**Transparencia Total:** Mostrar todo sobre esta transacción

**Wireframe para Escritorio:**

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

**Pestaña 2: Datos Crudos (lo que decía el PDF):**

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

**Pestaña 3: Decisiones de Normalización:**

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

**Pestaña 4: PDF Fuente:**

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

**Acciones del Usuario:**

1. **Editar categoría**
   - Clic en [Editar] junto a Categoría
   - Se abre menú desplegable con lista de categorías
   - Seleccionar nueva categoría → Se guarda inmediatamente
   - Notificación toast: "Categoría actualizada a Libros y Educación"
   - Se crea entrada en registro de auditoría

2. **Agregar nota**
   - Clic en campo de Notas
   - Escribir "Dividido con compañero de cuarto - me debe $43.71"
   - Clic fuera del campo → Se auto-guarda
   - La nota aparece inmediatamente

3. **Agregar etiqueta**
   - Clic en "[+ Agregar etiqueta]"
   - Escribir "reembolsable"
   - Presionar Enter → Etiqueta agregada
   - Se pueden agregar múltiples etiquetas

4. **Eliminar transacción**
   - Clic en [Eliminar Transacción]
   - Diálogo de confirmación: "¿Estás seguro? Esto no se puede deshacer."
   - El usuario confirma → Transacción eliminada suavemente (active=false)
   - Removida de la vista de lista

---

## Pantalla 4: Gestor de Categorías

**CRUD Simple:** Crear, leer, actualizar, eliminar categorías

**Wireframe para Escritorio:**

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

**Diálogo para Crear Nueva Categoría:**

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

**Acciones del Usuario:**

1. **Crear subcategoría**
   - Clic en [+ Nueva Categoría]
   - Nombre: "Cafeterías"
   - Padre: "Comida y Restaurantes"
   - Color: Naranja
   - Clic en [Crear] → Categoría agregada al árbol

2. **Editar categoría**
   - Clic en [Editar] junto a "Supermercado"
   - Cambiar nombre, color o categoría padre
   - Clic en [Guardar] → Se actualiza inmediatamente

3. **Eliminar categoría**
   - Clic en [Eliminar] junto a "Cafeterías"
   - Advertencia: "Esta categoría es usada por 10 transacciones. ¿Qué debemos hacer?"
   - Opciones:
     - Mover transacciones a categoría padre (Comida y Restaurantes)
     - Asignar a categoría diferente
     - Dejar sin categorizar
   - El usuario selecciona opción → Categoría eliminada, transacciones actualizadas

4. **Archivar categoría**
   - Para categorías ya no usadas pero con datos históricos
   - Clic en [Archivar] → Oculta de menús desplegables pero preservada
   - Alternar [Mostrar Archivadas] para ver

---

## Pantalla 5: Dashboard (3 Gráficos)

**Resumen de Un Vistazo:** Resumen mensual de gastos

**Wireframe para Escritorio:**

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

**Wireframe para Móvil:**

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

**Acciones del Usuario:**

1. **Cambiar período de tiempo**
   - Clic en "[Este Mes ▼]"
   - Opciones: Este Mes, Mes Pasado, Últimos 3 Meses, Últimos 6 Meses, Este Año, Personalizado
   - Seleccionar "Últimos 3 Meses" → Todos los gráficos se actualizan

2. **Profundizar en categoría**
   - Clic en barra "Supermercado $520"
   - Abre lista de transacciones filtrada mostrando solo transacciones de supermercado
   - Breadcrumb: Dashboard > Supermercado > Octubre 2024

3. **Exportar PDF**
   - Clic en [Exportar PDF]
   - Genera instantánea en PDF del dashboard con todos los gráficos
   - Descarga: `finance-dashboard-octubre-2024.pdf`

4. **Ver todas las categorías**
   - Clic en [Ver Todas las Categorías]
   - Abre desglose completo de categorías con subcategorías

---

## Flujo de Navegación

```
Inicio/Dashboard
  ├─ Subir Estado de Cuenta → Procesando → Éxito → Dashboard
  ├─ Transacciones
  │    └─ Clic en Transacción → Modal de Detalle
  │         ├─ Editar Categoría → Gestor de Categorías
  │         ├─ Ver Datos Crudos → Pestaña de Datos Crudos
  │         └─ Ver PDF Fuente → Visor de PDF
  ├─ Categorías → Gestor de Categorías
  │    └─ Crear/Editar/Eliminar Categorías
  └─ Configuración
       ├─ Cuentas
       ├─ Reglas (categorización)
       └─ Perfil
```

---

## Principios Clave de Interfaz de Usuario

**1. Acciones de Un Clic**
- La subida debe ser de un clic, no un asistente
- Los cambios de categoría se guardan inmediatamente (sin botón "Guardar")
- Confirmaciones mínimas (solo para acciones destructivas)

**2. Acciones Contextuales**
- Las acciones aparecen donde se necesitan (botón editar junto a categoría)
- Ocultar características avanzadas hasta que el usuario las necesite

**3. Divulgación Progresiva**
- Mostrar datos canónicos primero (lo que le importa al usuario)
- Datos crudos y decisiones disponibles via pestañas (para usuarios avanzados)

**4. Responsivo**
- Escritorio: Vista de tabla (más datos visibles)
- Móvil: Vista de tarjetas (más fácil de desplazar)
- Objetivos táctiles amigables (mínimo 44px)

**5. Retroalimentación Inmediata**
- Los cambios aparecen instantáneamente (UI optimista)
- Sincronización en segundo plano (no bloquear al usuario)
- Notificaciones toast para confirmaciones

---

## Estados de Carga

**Carga Inicial de Página:**
```
┌──────────────────────────────────────┐
│ Cargando transacciones...            │
│ ▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░ 35%       │
└──────────────────────────────────────┘
```

**Skeleton Loaders (preferidos sobre spinners):**
```
┌────────────────────────────────────────┐
│ ▓▓▓▓▓  ▓▓▓▓▓▓▓▓▓▓▓▓   ▓▓▓▓▓  ▓▓▓▓▓   │
│ ▓▓▓▓▓  ▓▓▓▓▓▓▓▓▓▓▓▓   ▓▓▓▓▓  ▓▓▓▓▓   │
│ ▓▓▓▓▓  ▓▓▓▓▓▓▓▓▓▓▓▓   ▓▓▓▓▓  ▓▓▓▓▓   │
└────────────────────────────────────────┘
```

---

## Estados Vacíos

**Sin Transacciones:**
```
┌────────────────────────────────────────┐
│                                        │
│           📄                           │
│     Sin transacciones aún              │
│                                        │
│  Sube tu primer estado de cuenta       │
│  para comenzar.                        │
│                                        │
│       [Subir Estado de Cuenta]         │
│                                        │
└────────────────────────────────────────┘
```

**Sin Resultados de Búsqueda:**
```
┌────────────────────────────────────────┐
│ No hay transacciones que coincidan    │
│ con "starbuck"                         │
│                                        │
│ Intenta:                               │
│ • Revisar tu ortografía                │
│ • Usar palabras clave diferentes       │
│ • Limpiar filtros                      │
│                                        │
│       [Restablecer Filtros]            │
└────────────────────────────────────────┘
```

---

## Resumen

**Esta interfaz de usuario es:**
- ✅ Simple (5 pantallas principales)
- ✅ Intuitiva (patrones estándar)
- ✅ Responsiva (escritorio + móvil)
- ✅ Rápida (actualizaciones optimistas)

**Esta interfaz de usuario evita:**
- ❌ Asistentes complejos (formularios de múltiples pasos)
- ❌ Modales excesivos (solo para detalles)
- ❌ Características ocultas (todo es descubrible)
- ❌ Jerga (lenguaje claro)
