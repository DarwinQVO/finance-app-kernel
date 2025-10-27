# Flujos de Usuario Principales

> **Propósito:** Contar la historia de cómo los usuarios interactúan con la aplicación de finanzas a través de 11 flujos críticos

---

## Flujo 1: Cuando el Usuario Sube un Documento

**El Ritual del Domingo por la Mañana**

El usuario abre la aplicación de finanzas un domingo por la mañana con su extracto de Bank of America del mes pasado. Está listo para revisar sus gastos.

**La Experiencia de Carga:**

1. **Encontrando el archivo**
   - Hace clic en el botón "Subir Extracto"
   - Se abre el selector de archivos
   - Navega a ~/Downloads/bofa-statement-october.pdf
   - Selecciona el archivo

2. **El sistema lo acepta inmediatamente**
   - Sin menú desplegable preguntando "¿qué tipo de archivo es este?"
   - Sin diálogo de confirmación
   - Aparece barra de progreso: "Subiendo... 1.2 MB"
   - La carga se completa en 2 segundos

3. **Comienza el procesamiento**
   - La pantalla muestra: "Procesando extracto... esto puede tomar 30 segundos"
   - El usuario puede navegar a otra parte - el procesamiento ocurre en segundo plano
   - Aparecerá una notificación cuando esté completo

**Por qué esto importa:**
- El usuario no debería necesitar especificar "source_type=bofa_pdf" - el sistema debería detectarlo
- Pero para v1, preguntaremos porque la detección de formato PDF es compleja
- La carga debería sentirse instantánea aunque el análisis tome 30 segundos

**Qué puede salir mal:**
- El archivo no es un PDF → Mostrar error: "Por favor sube un archivo PDF"
- El archivo está corrupto → Mostrar error: "No se pudo leer el PDF - el archivo puede estar corrupto"
- El archivo ya fue subido → Mostrar advertencia: "Este archivo fue subido el 15 de octubre"

---

## Flujo 2: Cuando el Sistema Hace el Análisis

**Entre Bastidores (El usuario no ve esto)**

El PDF está en almacenamiento. Un worker toma el trabajo.

**El Flujo de Análisis:**

1. **Abriendo el PDF**
   - La biblioteca PyPDF2 abre el archivo
   - Escanea el contenido de texto
   - Identifica la estructura de tabla en la página 2

2. **Extrayendo transacciones**
   - Encuentra el encabezado de la tabla de transacciones: "Date | Description | Amount"
   - Lee cada fila
   - Extrae 42 transacciones
   - Datos crudos: fechas como "10/15/2024", montos como "$87.43", descripciones como "WHOLE FOODS MARKET #1234"

3. **Guardando observaciones crudas**
   - Cada transacción guardada como "observación" (cruda, sin procesar)
   - Campos: date_raw="10/15/2024", description_raw="WHOLE FOODS MARKET #1234", amount_raw="$87.43"
   - Cambios de estado: queued_for_parse → parsing → parsed

**La Decisión Crítica:**
- Almacenar datos TAL CUAL, no limpiarlos todavía
- ¿Por qué? Si descubrimos que el limpiador tiene un bug, podemos reprocesar desde los datos crudos
- Las observaciones crudas son inmutables - nunca modificadas

**Casos Extremos:**
- PDF tiene 0 transacciones → Estado: parsed (con advertencia, no error)
- PDF tiene 1000+ transacciones → Funciona bien, solo toma más tiempo
- Formato de PDF cambió (BoFA rediseñó su extracto) → El parser falla, usuario recibe error claro

---

## Flujo 3: Cuando el Usuario Navega OL (Capa Objetiva)

**La Experiencia de Exploración**

El usuario ve su lista de transacciones. Una transacción llama su atención:

```
Oct 15    WHOLE FOODS MARKET    -$87.43    Groceries
```

**Hace clic en ella. Se abre un modal mostrando:**

**Pestaña 1: Datos Canónicos (Lo que determinamos que es verdad)**
```
Fecha:         15 de octubre, 2024
Comerciante:   Whole Foods Market
Monto:         -87.43 USD
Categoría:     Groceries
Cuenta:        Chase Checking (****1234)
Confianza:     Alta (0.95)
```

**Pestaña 2: Observación Cruda (Lo que decía el PDF)**
```
Fecha:         "10/15/2024"
Descripción:   "WHOLE FOODS MARKET #1234 SAN FRANCISCO CA"
Monto:         "$87.43"
Cuenta:        "****1234"
```

**Pestaña 3: Decisiones de Normalización (Cómo pasamos de crudo → canónico)**
```
Limpieza de Fecha:
  Crudo: "10/15/2024" → Canónico: 2024-10-15 (ISO 8601)
  Regla: Analizar formato MM/DD/YYYY

Normalización de Comerciante:
  Crudo: "WHOLE FOODS MARKET #1234 SAN FRANCISCO CA"
  → Canónico: "Whole Foods Market"
  Regla: Eliminar número de tienda y ubicación

Análisis de Monto:
  Crudo: "$87.43" → Canónico: -87.43
  Regla: Remover "$", negar (es un gasto)

Asignación de Categoría:
  Comerciante: "Whole Foods Market" → Categoría: "Groceries"
  Regla: Coincidencia de patrón de comerciante (configurado en YAML)
```

**Pestaña 4: Documento Fuente**
- Botón [Ver PDF] → Abre bofa-statement-october.pdf en página 2
- Transacción resaltada en amarillo

**Por qué esto importa:**
- Transparencia completa - el usuario puede ver exactamente cómo procesamos sus datos
- Si algo se ve mal, pueden ver de dónde vino
- Genera confianza

---

## Flujo 4: Cuando el Usuario Navega RL (Capa de Representación)

**La Vista del Dashboard**

El usuario quiere ver sus tendencias de gasto. Navega al Dashboard.

**Lo que ve:**

```
┌─────────────────────────────────────────────────┐
│ Resumen de Octubre 2024                         │
├─────────────────────────────────────────────────┤
│ Ingresos:    $4,200  ↑ 5% vs Sept               │
│ Gastos:      $3,100  ↓ 2% vs Sept               │
│ Ahorro Neto: $1,100  ↑ 12% vs Sept              │
├─────────────────────────────────────────────────┤
│ Principales Categorías de Gasto:                │
│                                                 │
│ 🏠 Renta          $1,200  ████████████ (39%)   │
│ 🍔 Groceries        $520  ████         (17%)   │
│ 🚗 Transporte       $380  ███          (12%)   │
│ ⚡ Servicios        $250  ██           (8%)    │
│ 🎬 Entretenimiento  $180  █            (6%)    │
└─────────────────────────────────────────────────┘
```

**Elementos Interactivos:**
- Clic en "Renta" → Profundiza en todas las transacciones de renta
- Clic en "Groceries" → Muestra desglose por tienda (Whole Foods, Safeway, Trader Joe's)
- Selector de fecha → Cambiar el mes visualizado

**Por qué esto es RL (Capa de Representación):**
- Es una VISTA construida sobre datos de OL (Capa Objetiva)
- Las transacciones canónicas no han cambiado
- Solo las estamos presentando de manera diferente
- Mismos datos, perspectiva diferente

---

## Flujo 5: Cuando el Sistema Detecta Duplicado (Apple Card)

**El Problema de Deduplicación**

El usuario sube su extracto de octubre. El sistema comienza a procesar.

**Lo que sucede:**

1. **Sistema lee transacción:** "APPLE.COM/BILL $14.99" el 15 de octubre
2. **Cálculo de hash:** Calcula hash de (fecha + comerciante + monto + cuenta)
3. **Verificación de duplicado:** Busca transacción existente con el mismo hash
4. **Se encontró coincidencia:** Transacción idéntica del 15 de octubre ya existe (subida hace 2 semanas desde PDF diferente)

**Decisión del Sistema:**
```
Estado: DUPLICADO DETECTADO
Acción: Omitir creación de transacción canónica
Razón: Colisión de hash - misma transacción ya en sistema
Original: TX_abc123 (subida el 16 de oct desde statement-september.pdf)
Duplicado: TX_xyz789 (carga actual desde statement-october.pdf)
```

**Lo que ve el usuario:**
- Resumen de importación: "42 transacciones procesadas, 1 duplicado omitido"
- Detalles disponibles si hacen clic en "Mostrar duplicados"

**Por qué esto importa:**
- Los bancos a veces incluyen transacciones del mes anterior en extractos nuevos
- Tarjetas de crédito muestran transacciones pendientes en múltiples extractos
- Sin deduplicación, el usuario vería cargos duplicados

**El Caso Extremo:**
¿Qué pasa si dos cargos idénticos ocurrieron legítimamente el mismo día?
- Ejemplo: Dos compras de $4.75 en Starbucks el 15 de octubre
- El hash sería idéntico
- Solución actual: El usuario puede "des-fusionar" manualmente si es necesario
- Futuro: Usar hora de transacción (HH:MM:SS) si está disponible

---

## Flujo 6: Cuando el Sistema Hace Resolución de Cuenta

**El Problema de Coincidencia de Cuenta**

El PDF dice: "Account: ****1234"

¿Pero cuál cuenta es esa? El usuario tiene 6 cuentas.

**Flujo de Resolución:**

1. **Sistema lee observación cruda:**
   - Identificador de cuenta: "****1234"
   - Desde PDF: bofa-statement-october.pdf

2. **Búsqueda en Registro de Cuentas:**
   ```
   Cuentas Registradas:
   - Chase Checking (****1234)     ← ¡COINCIDENCIA!
   - Chase Savings (****5678)
   - BoFA Checking (****9012)
   - BoFA Credit Card (****3456)
   - Apple Card (****7890)
   - Amex (****2345)
   ```

3. **Coincidencia encontrada:**
   - Transacción canónica obtiene: account_id = "ACC_chase_checking"
   - Enlace establecido: Transacción → Cuenta

**¿Qué pasa si no hay coincidencia?**
- Estado: NO RESUELTA
- Usuario ve: "Cuenta desconocida ****1234 - por favor mapear a cuenta existente"
- Usuario crea mapeo: "****1234" → "Chase Checking"
- Sistema reprocesa: Todas las transacciones con ****1234 ahora enlazadas correctamente

**Por qué el Registro de Cuentas es cerrado:**
- Usuario crea cuentas explícitamente (no auto-generadas desde extractos)
- Previene "explosión de cuentas" (100 cuentas por errores tipográficos)
- Asegura datos limpios

---

## Flujo 7: Cuando el Sistema Hace Resolución de Contraparte

**El Problema de Coincidencia de Comerciante**

Descripción cruda: "WHOLE FOODS MARKET #1234 SAN FRANCISCO CA"

¿Quién es la contraparte? ¿Cuál es el nombre canónico del comerciante?

**Flujo de Resolución:**

1. **Sistema lee descripción cruda:**
   - "WHOLE FOODS MARKET #1234 SAN FRANCISCO CA"

2. **Reglas de Normalización Aplicadas:**
   ```yaml
   # merchantnormalization_rules.yaml
   - pattern: "WHOLE FOODS MARKET #\d+"
     canonical: "Whole Foods Market"
     strip: ["#\d+", "SAN FRANCISCO CA"]
   ```

3. **Búsqueda de Contraparte:**
   - Buscar contrapartes existentes para "Whole Foods Market"
   - **Coincidencia encontrada:** counterparty_id = "CP_wholefoods"

4. **Si no hay coincidencia → Crear nueva contraparte:**
   ```
   Nombre Canónico: "Whole Foods Market"
   Alias: ["WHOLE FOODS MARKET #1234", "WFM", "WHOLE FOODS"]
   Tipo: merchant
   Categoría: groceries
   ```

**Por qué el Registro de Contrapartes es abierto:**
- Auto-crea desde nombres de comerciantes
- Aprende alias con el tiempo
- Usuario puede fusionar duplicados después

**El Ejemplo de Coincidencia Difusa:**
- Transacción 1: "WHOLE FOODS MARKET #1234"
- Transacción 2: "WHOLE FOODS MKT #5678"
- Transacción 3: "WFM SAN FRANCISCO"

Sistema crea 3 contrapartes inicialmente. Usuario ve sugerencia de duplicado:
"Estos se ven similares - ¿fusionarlos?"
→ Usuario confirma → Sistema los fusiona en una contraparte canónica

---

## Flujo 8: Cuando el Sistema Hace Clustering (Post-Resolución de Entidad)

**El Problema de Descubrimiento de Patrones**

Después de la resolución de cuenta y contraparte, el sistema busca patrones.

**Flujo de Clustering:**

1. **Encontrar transacciones similares:**
   ```
   Grupo 1: Compras en Starbucks
   - 1 Oct:  STARBUCKS #123 SF    $4.75
   - 5 Oct:  STARBUCKS #456 OAK   $5.25
   - 12 Oct: STARBUCKS #123 SF    $4.75
   - 20 Oct: STARBUCKS #789 NYC   $6.50

   Características comunes:
   - Contraparte: Starbucks
   - Categoría: Coffee
   - Rango de monto: $4-7
   - Frecuencia: Semanal
   ```

2. **Sistema sugiere:**
   ```
   Cluster: "Café Semanal"
   Miembros: 4 transacciones
   Promedio: $5.31
   Confianza: Alta (0.89)
   ```

3. **El usuario puede:**
   - Aceptar cluster → Transacciones obtienen etiqueta "cafe-semanal"
   - Rechazar cluster → Sistema no sugerirá de nuevo
   - Modificar cluster → Agregar/remover transacciones

**Por qué el clustering importa:**
- Descubre patrones de gasto automáticamente
- Ayuda al usuario a entender hábitos
- Habilita presupuesto por patrón

**El Umbral:**
- Mínimo 3 transacciones para formar cluster
- Puntuación de similitud > 0.7 (configurable)
- Ventana de tiempo: últimos 90 días

---

## Flujo 9: Cuando el Usuario Corrige un Error

**El Descubrimiento del Error**

El usuario nota: "AMZN MKTPLACE" el 10 de octubre por $45.99 está categorizado como "Shopping"

Pero en realidad fue una compra de libro - debería ser "Books & Education"

**Flujo de Corrección:**

1. **Usuario hace clic en transacción**
2. **Hace clic en "Editar Categoría"**
3. **Desplegable muestra:**
   ```
   Actual: Shopping (de regla de auto-categorización)

   Cambiar a:
   ● Books & Education
   ○ Shopping
   ○ Entertainment
   ○ Other
   ```

4. **Usuario selecciona "Books & Education" y hace clic en Guardar**

**Lo que sucede entre bastidores:**

```
1. Sistema crea override:
   {
     transaction_id: "TX_abc123",
     field: "category",
     old_value: "Shopping",
     new_value: "Books & Education",
     reason: "manual_correction",
     corrected_by: "user",
     corrected_at: "2024-10-27T10:30:00Z"
   }

2. Transacción canónica actualizada:
   category = "Books & Education"

3. Entrada de proveniencia creada:
   "Transaction TX_abc123 category changed from Shopping to Books & Education
    by user at 2024-10-27T10:30:00Z"
```

**Usuario ve:**
- Transacción ahora muestra "Books & Education"
- Pequeño indicador: "Editado" (con tooltip mostrando valor original)

**¿Pueden revertirse las correcciones?**
- ¡Sí! Usuario hace clic en "Ver Historial" → Ve todos los cambios → "Revertir a Original"
- Sistema restaura "Shopping" y registra la reversión

**Por qué esto importa:**
- Auto-categorización no es perfecta
- Usuario debería poder corregir errores
- Rastro de auditoría completo de cambios

---

## Flujo 10: Cuando el Sistema Conecta Transacción a Serie

**El Patrón de Pago Recurrente**

El usuario tiene Netflix: $14.99 cada mes el día 15.

**Detección de Patrón:**

1. **Sistema ve transacciones:**
   ```
   15 Sep: NETFLIX.COM    $14.99
   15 Oct: NETFLIX.COM    $14.99
   15 Nov: NETFLIX.COM    $14.99
   ```

2. **Patrón reconocido:**
   ```
   Serie Detectada:
   - Nombre: Suscripción Netflix
   - Monto: $14.99 (± $1 tolerancia)
   - Frecuencia: Mensual
   - Día: 15 del mes
   - Contraparte: Netflix
   - Confianza: Alta (0.95)
   ```

3. **Sistema crea Serie:**
   ```
   series_id: "SER_netflix"
   template: {
     amount: 14.99,
     counterparty: "Netflix",
     category: "Entertainment",
     recurrence: "monthly"
   }
   ```

4. **Enlaza transacciones a serie:**
   ```
   TX_sep15 → SER_netflix (instancia 1)
   TX_oct15 → SER_netflix (instancia 2)
   TX_nov15 → SER_netflix (instancia 3)
   ```

**Experiencia del Usuario:**
- Transacción muestra insignia: "Recurrente: Suscripción Netflix"
- Dashboard muestra: "Suscripciones Mensuales: $89.94" (suma de todas las recurrentes)
- Puede hacer clic para ver todas las instancias

**Detección de Varianza:**
- Si el cargo de diciembre es $15.99 en lugar de $14.99
- Sistema marca: "Cargo de Netflix fue $1 más alto de lo usual - ¿aumento de precio?"
- Usuario puede aceptar nuevo monto o investigar

**Detección de Pago Faltante:**
- Si el 15 de enero pasa sin cargo de Netflix
- Sistema alerta: "Falta pago esperado de Netflix"
- Usuario puede marcar como "suscripción cancelada" o "aún pendiente"

---

## Flujo 11: Cuando el Sistema Categoriza para Impuestos USA

**La Preparación de Temporada de Impuestos**

1 de abril. El usuario necesita preparar sus impuestos.

**Categorización Fiscal:**

1. **Sistema tiene transacciones categorizadas:**
   ```
   Groceries: $6,240
   Renta: $14,400
   Servicios: $3,000
   Salud: $2,500
   Donaciones Caritativas: $1,200
   ```

2. **Mapeo de categoría fiscal aplicado:**
   ```yaml
   # Reglas Fiscales USA
   Deducible:
     - Gastos Médicos: Salud ($2,500)
     - Caridad: Donaciones Caritativas ($1,200)

   No Deducible:
     - Personal: Groceries, Renta, Servicios ($23,640)
   ```

3. **Reporte fiscal generado:**
   ```
   Schedule A (Deducciones Detalladas):
   - Médico: $2,500
   - Caridad: $1,200
   Total Detallado: $3,700

   Deducción Estándar: $13,850
   Recomendación: Tomar deducción estándar
   ```

**Por qué la categoría importa para impuestos:**
- Gastos médicos solo deducibles si > 7.5% AGI
- Donaciones caritativas requieren recibos
- Gastos de negocio (si auto-empleado) completamente deducibles

**El Problema Multi-Jurisdicción:**
- El usuario vive en California pero trabajó remotamente desde México por 3 meses
- Algunas transacciones en MXN (Pesos Mexicanos)
- Necesita rastrear qué ingresos/gastos aplican a qué jurisdicción

**Solución:**
- Transacciones obtienen etiquetas de jurisdicción: "USA" o "Mexico"
- Reportes fiscales separados por jurisdicción
- Conversión FX aplicada a tasas de fecha de transacción

---

## Flujo 12: Cuando el Sistema Enlaza Transacción de Comisión FX

**La Comisión Oculta de Cambio de Divisa**

El usuario hizo una compra en Europa:

```
10 Oct: RESTAURANT PARIS    -€45.00
10 Oct: FOREIGN FX FEE      -$2.50
```

**La Conexión:**

1. **Sistema ve dos transacciones el mismo día:**
   - Transacción 1: €45.00 (convertido a $49.50 a tasa 1.10)
   - Transacción 2: $2.50 (comisión FX)

2. **Coincidencia de patrón:**
   ```
   Criterios para enlace de comisión FX:
   - Misma fecha (o día siguiente)
   - Descripción contiene "FX FEE", "FOREIGN", "EXCHANGE"
   - Monto < 5% de transacción extranjera
   - Cuenta: Misma tarjeta de crédito
   ```

3. **Relación creada:**
   ```
   type: "foreign_exchange_fee"
   primary: TX_paris (€45.00)
   fee: TX_fxfee ($2.50)
   total_cost: $52.00
   effective_rate: 1.156 (incluyendo comisión)
   ```

**Experiencia del Usuario:**
- Transacción primaria muestra: "€45.00 ($49.50 + $2.50 comisión FX = $52.00 total)"
- Transacción de comisión FX muestra: "Comisión por: transacción Restaurant Paris"
- Ambas enlazadas visualmente en UI

**Por qué esto importa:**
- Compañías de tarjetas de crédito ocultan comisiones FX como transacciones separadas
- Usuario necesita ver el costo real de compras internacionales
- Afecta precisión de análisis de gasto

**Caso Extremo - Viaje Multi-Moneda:**
El usuario viaja a Europa por 2 semanas. 30 transacciones en EUR, 5 comisiones FX.

El sistema debe:
1. Emparejar cada comisión FX a transacción correcta (por heurística de monto)
2. Manejar comisiones divididas (una comisión FX para múltiples transacciones pequeñas)
3. Permitir enlace manual si auto-emparejamiento falla

---

## Resumen: El Mapa de Flujos de Usuario

```
1. Subir Documento → Sistema almacena, pone en cola para análisis
2. Sistema Analiza → Extrae observaciones crudas
3. Usuario Navega OL → Transparencia completa en procesamiento
4. Usuario Navega RL → Insights y visualizaciones
5. Detección de Duplicados → Previene doble conteo
6. Resolución de Cuenta → Enlaza a cuentas del usuario
7. Resolución de Contraparte → Identifica comerciantes
8. Clustering → Descubre patrones de gasto
9. Corrección del Usuario → Corrige errores de categorización
10. Conexión de Serie → Identifica pagos recurrentes
11. Categorización Fiscal → Prepara para temporada de impuestos
12. Enlace de Comisión FX → Muestra costo real de compras internacionales
```

**Principio Clave:**
El sistema debe ser inteligente pero transparente. Cada decisión debe ser explicable y corregible por el usuario.
