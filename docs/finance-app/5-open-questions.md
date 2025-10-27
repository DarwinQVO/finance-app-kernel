# Preguntas Abiertas

> **Propósito:** Documentar características y decisiones de diseño no decididas que necesitan input del usuario

---

## Filosofía

**No todas las decisiones necesitan tomarse por adelantado.** Algunas características dependen de patrones de uso reales. Este documento rastrea preguntas que deben responderse basándose en experiencia del mundo real, no especulación.

**Proceso de Decisión:**
1. Construir v1 sin la característica
2. Usar la app por 1-2 meses
3. Si emerge un punto de dolor → Revisitar pregunta
4. Si no hay punto de dolor → Característica no necesaria

---

## Pregunta 1: Soporte Multi-Moneda

**La Pregunta:**
¿Debería la app soportar transacciones en múltiples monedas (EUR, GBP, MXN, etc.)?

**Estado Actual (v1):**
- Todas las transacciones se asumen en USD
- Sin campo `currency_code`
- Sin seguimiento de tasa de cambio
- Transacciones en moneda extranjera muestran solo monto convertido

**Escenario de Ejemplo:**
El usuario viaja a Europa. El estado de cuenta de tarjeta de crédito muestra:
```
10/15/2024  RESTAURANT PARIS EUR 45.00
            EXCHANGE RATE 1.10              -$49.50
10/15/2024  FOREIGN TRANSACTION FEE          -$2.50
```

**Manejo actual:**
- Importar como dos transacciones USD: -$49.50 y -$2.50
- Se pierde información original de EUR 45.00
- No se puede calcular tasa de cambio efectiva incluyendo comisión

**Lo que multi-moneda habilitaría:**
- Almacenar moneda original: `amount=45.00, currency=EUR`
- Almacenar monto convertido: `amount_usd=49.50`
- Rastrear tasa de cambio: `rate=1.10`
- Calcular tasa efectiva: `(49.50 + 2.50) / 45.00 = 1.156`
- Ver gastos en moneda original

**Criterios de Decisión:**

| Si... | Entonces... |
|-------|-------------|
| El usuario rara vez viaja internacionalmente (< 2x/año) | **No agregar** - Complejidad no vale la pena |
| El usuario viaja frecuentemente (> 4x/año) | **Agregarlo** - Característica útil |
| El usuario tiene ingresos/gastos extranjeros | **Agregarlo** - Esencial |
| El usuario solo gasta en USD | **No agregar** - No necesario |

**Impacto en Modelo de Datos:**
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

**Recomendación:**
**COMENZAR SIN ESTO.** Agregar solo si el usuario viaja frecuentemente.

---

## Pregunta 2: Estrategia de Auto-Categorización

**La Pregunta:**
¿Debería la categorización usar:
- **Opción A:** Coincidencia simple de patrones (reglas YAML)
- **Opción B:** Aprendizaje automático (entrenar en historial del usuario)
- **Opción C:** Híbrido (reglas + ML)

**Estado Actual (v1):**
Usando **Opción A** - Coincidencia de patrones con reglas YAML:
```yaml
- pattern: "WHOLE FOODS"
  category: "Groceries"
  confidence: 0.9
```

**Pros de Coincidencia de Patrones:**
- ✅ Simple de entender
- ✅ Determinístico (mismo input → mismo output)
- ✅ Fácil de debugear (solo mirar archivo YAML)
- ✅ El usuario puede editar reglas directamente
- ✅ No se necesitan datos de entrenamiento

**Contras de Coincidencia de Patrones:**
- ❌ Requiere creación manual de reglas
- ❌ No puede aprender de correcciones del usuario
- ❌ No se adapta a nuevos comerciantes
- ❌ Se necesitan 50-100 reglas para buena cobertura

**Lo que ML habilitaría:**
- Aprender de correcciones: Usuario cambia "Amazon" de "Compras" a "Libros" → Sistema aprende
- Adaptativo: Precisión mejora con el tiempo
- Manejar nuevos comerciantes automáticamente

**Lo que ML requeriría:**
- Datos de entrenamiento: Necesita 100+ transacciones categorizadas mínimo
- Complejidad: scikit-learn, entrenamiento de modelo, puntajes de confianza
- Explicabilidad: Más difícil responder "¿por qué categorizaste esto como X?"

**Criterios de Decisión:**

| Si... | Entonces... |
|-------|-------------|
| El usuario tiene < 500 transacciones | **Coincidencia de patrones** - No suficientes datos de entrenamiento |
| El usuario se siente cómodo editando YAML | **Coincidencia de patrones** - Usuario prefiere control |
| Precisión de categorización > 85% con reglas | **Coincidencia de patrones** - Suficientemente bueno |
| El usuario corrige frecuentemente categorías (> 10% de transacciones) | **Probar ML** - Reglas no funcionan |
| El usuario quiere característica "auto-mejorar" | **Probar ML** - Vende la característica |

**Enfoque Híbrido (Opción C):**
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

**Recomendación:**
**COMENZAR CON REGLAS.** Medir tasa de corrección. Si > 10%, considerar ML.

---

## Pregunta 3: Transacciones Divididas

**La Pregunta:**
¿Deberían los usuarios poder dividir una única transacción entre múltiples categorías?

**Escenario de Ejemplo:**
El usuario compra en Target:
```
10/15/2024  TARGET #1234               -$127.50
```

Pero los $127.50 realmente incluyen:
- Supermercado: $45.00
- Artículos del hogar: $35.50
- Ropa: $47.00

**Estado Actual (v1):**
- Transacción asignada a UNA categoría solamente
- El usuario puede agregar nota: "Incluye supermercado + hogar + ropa"
- No hay manera de dividir en múltiples categorías

**Lo que división habilitaría:**
- Desglose preciso de gastos por categoría
- Porciones deducibles de impuestos (si compras en Target incluyen suministros de oficina)
- Seguimiento de presupuesto por categoría

**Lo que división requeriría:**
- UI para crear divisiones (% o montos en $)
- Validación: Divisiones deben sumar al total
- Consultas complejas: "Mostrar todos los gastos de supermercado" → Incluir transacciones parciales
- Complejidad de exportación: ¿La división se muestra como 1 fila o 3 filas?

**Impacto en Modelo de Datos:**
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

**Criterios de Decisión:**

| Si... | Entonces... |
|-------|-------------|
| El usuario rara vez compra en tiendas con múltiples departamentos (< 5% de transacciones) | **No agregar** - Caso especial |
| El usuario necesita seguimiento de deducción de impuestos para compras mixtas | **Agregarlo** - Requerido para impuestos |
| El usuario se siente cómodo con divisiones aproximadas de categoría | **No agregar** - Campo de notas suficiente |
| Seguimiento de presupuesto requiere divisiones exactas | **Agregarlo** - Afecta presupuestos |

**Recomendación:**
**COMENZAR SIN ESTO.** Usar campo de notas. Agregar si emergen requisitos de impuestos.

---

## Pregunta 4: Adjuntos de Recibos

**La Pregunta:**
¿Deberían los usuarios poder adjuntar imágenes/PDFs de recibos a transacciones?

**Escenario de Ejemplo:**
- Gasto de negocio: Necesita recibo para reembolso
- Compra grande: Garantía, política de devolución en recibo
- Deducción de impuestos: IRS requiere recibos para ciertos gastos

**Estado Actual (v1):**
- Sin soporte de adjuntos
- El usuario puede agregar nota: "Recibo en Dropbox: /tax-docs/2024/target-receipt-oct15.pdf"

**Lo que adjuntos habilitarían:**
- Acceso al recibo con un clic desde la transacción
- Almacenamiento de recibos integrado con la app
- OCR para extraer monto/fecha del recibo (validar contra transacción)

**Lo que adjuntos requerirían:**
- UI de subida de archivos (arrastrar y soltar imágenes)
- Almacenamiento (500 recibos × 1 MB = 500 MB)
- Visor de imágenes en modal de detalle de transacción
- Gestión de archivos (eliminar, descargar)

**Impacto en Modelo de Datos:**
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

**Criterios de Decisión:**

| Si... | Entonces... |
|-------|-------------|
| El usuario solo rastrea gastos personales | **No agregar** - No necesario |
| El usuario necesita recibos para reembolso de negocio | **Agregarlo** - Esencial para trabajo |
| El usuario necesita recibos para auditorías de impuestos | **Agregarlo** - Cumplimiento con IRS |
| El usuario se siente cómodo almacenando recibos externamente (Dropbox, Google Drive) | **No agregar** - Externo funciona |

**Recomendación:**
**COMENZAR SIN ESTO.** Agregar si emerge caso de uso de negocio/impuestos.

---

## Pregunta 5: Seguimiento de Presupuesto

**La Pregunta:**
¿Debería la app incluir seguimiento mensual de presupuesto (gasto planificado vs. real)?

**Escenario de Ejemplo:**
El usuario establece presupuestos mensuales:
- Supermercado: $500/mes
- Restaurantes: $200/mes
- Entretenimiento: $100/mes

La app muestra:
- Supermercado: $520 gastado (104% del presupuesto) ⚠️
- Restaurantes: $180 gastado (90% del presupuesto) ✅
- Entretenimiento: $85 gastado (85% del presupuesto) ✅

**Estado Actual (v1):**
- Sin seguimiento de presupuesto
- El usuario puede ver gasto total por categoría en dashboard
- Sin alertas o advertencias

**Lo que presupuestos habilitarían:**
- Alertas proactivas: "Has gastado 90% de tu presupuesto de Supermercado"
- Establecimiento de metas: "Ahorrar $500 este mes"
- Análisis de tendencias: "Supermercado sobre presupuesto 3 meses seguidos"

**Lo que presupuestos requerirían:**
- UI de configuración de presupuesto (establecer monto por categoría)
- Barras de progreso de presupuesto en dashboard
- Sistema de alertas (email/push cuando se excede presupuesto)
- Comparación histórica presupuesto vs. real

**Impacto en Modelo de Datos:**
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

**Criterios de Decisión:**

| Si... | Entonces... |
|-------|-------------|
| El usuario quiere seguimiento pasivo (solo ver gastos) | **No agregar** - Dashboard suficiente |
| El usuario quiere control activo (mantenerse dentro del presupuesto) | **Agregarlo** - Característica central |
| El usuario tiene ingresos irregulares (freelancer, contratista) | **Tal vez agregar** - Presupuestos menos útiles |
| El usuario tiene ingresos y gastos fijos | **Agregarlo** - Muy útil |

**Recomendación:**
**COMENZAR SIN ESTO.** Usar app por 1-2 meses. Si el usuario repetidamente pregunta "¿gasté de más en X?" → Agregarlo.

---

## Pregunta 6: Seguimiento de Transacciones Recurrentes

**La Pregunta:**
¿Qué nivel de soporte de transacciones recurrentes?
- **Opción A:** Manual (usuario crea serie, vincula transacciones manualmente)
- **Opción B:** Auto-detectar (sistema sugiere series basándose en patrones)
- **Opción C:** Proactivo (sistema crea series automáticamente, usuario aprueba)

**Estado Actual (v1):**
Implementando **Opción B** - Auto-detectar con sugerencias

**Cómo funciona:**
- Sistema detecta: Netflix $14.99 cada mes el día 15
- Usuario ve notificación: "Parece que Netflix es un pago recurrente - ¿rastrearlo?"
- Usuario aprueba → Serie creada
- Cargos futuros de Netflix se auto-vinculan a la serie

**Preguntas Abiertas:**

**P6.1: Alertas de Pago Faltante**
Si el 15 de noviembre pasa sin cargo de Netflix:
- **¿Alertar inmediatamente?** "Falta el pago esperado de Netflix"
- **¿Esperar 5 días?** (el pago podría retrasarse)
- **¿Configurable por usuario?** Dejar que usuario establezca período de gracia por serie

**Decisión:**
Esperar 5 días por defecto. Agregar configuración de usuario si es necesario.

**P6.2: Tolerancia de Varianza**
El cargo de Netflix es usualmente $14.99, pero diciembre muestra $15.99:
- **¿Rechazar como no coincidente?** (crear nueva transacción, no vincular a serie)
- **¿Aceptar con advertencia?** "Cargo de Netflix fue $1 más alto - ¿aumento de precio?"
- **¿Auto-actualizar monto de serie?** (asumir nuevo precio hacia adelante)

**Decisión:**
Aceptar con advertencia. Usuario revisa y:
- Confirma aumento de precio → Actualizar monto de serie a $15.99
- Rechaza → Varianza única, mantener serie en $14.99

**P6.3: Cancelación de Serie**
Usuario cancela Netflix. ¿Cómo debería manejar el sistema?
- **¿Auto-detectar?** Si 2 meses pasan sin cargo, sugerir "¿Marcar serie como cancelada?"
- **¿Solo manual?** Usuario debe cancelar serie explícitamente
- **¿Mantener para siempre?** Serie permanece activa indefinidamente (alerta al usuario cada mes)

**Decisión:**
Auto-sugerir después de 2 pagos faltantes. No auto-cancelar (usuario podría reanudar suscripción).

---

## Pregunta 7: Integración de Preparación de Impuestos

**La Pregunta:**
¿Qué tan profundo debería ir el soporte de impuestos?

**Niveles de Soporte:**

**Nivel 1: Básico (v1)**
- Categorizar transacciones
- Etiquetar gastos deducibles de impuestos
- Exportar CSV para contador

**Nivel 2: Intermedio**
- Formularios de impuestos pre-llenados (Schedule C para auto-empleados)
- Soporte multi-jurisdicción (vivió en 2 estados)
- Seguimiento de millaje (si gastos de auto deducibles)

**Nivel 3: Avanzado**
- Exportación directa a TurboTax, H&R Block
- Cálculo estimado de impuestos en tiempo real
- Recordatorios de pagos de impuestos trimestrales
- Documentación de rastro de auditoría

**Estado Actual (v1):**
Implementando **Nivel 1** - Categorización básica

**Preguntas Abiertas:**

**P7.1: ¿Qué es deducible?**
Usuario marca transacción como "deducible de impuestos" pero:
- Gastos médicos solo deducibles si > 7.5% AGI
- Deducción de oficina en casa tiene reglas complejas
- Comidas son 50% deducibles (para negocio)

**¿Debería la app:**
- **¿Confiar en usuario?** Dejarlos marcar cualquier cosa deducible (contador verificará)
- **¿Hacer cumplir reglas del IRS?** Bloquear marcar supermercado como deducible
- **¿Advertir al usuario?** "Deducciones médicas requieren > $X total para reclamar"

**Decisión:**
Confiar en usuario (v1). Agregar advertencias si usuario las solicita.

**P7.2: Gastos de Doble Propósito**
Usuario compra laptop para 60% trabajo, 40% personal:
- ¿Dividir transacción? (ver Pregunta 3)
- ¿Campo de porcentaje personalizado? `deductible_percentage=60%`
- ¿Solo notas? "60% uso de trabajo"

**Decisión:**
Solo notas (v1). Agregar campo de porcentaje si es caso de uso común.

---

## Pregunta 8: App Móvil vs. Solo Web

**La Pregunta:**
¿Debería haber una app móvil nativa (iOS/Android)?

**Estado Actual (v1):**
Solo app web (diseño responsivo funciona en navegador móvil)

**Lo que app móvil habilitaría:**
- **Captura de recibos:** Tomar foto → auto-adjuntar a transacción
- **Notificaciones push:** "Pago de Netflix debido mañana"
- **Modo offline:** Ver transacciones sin internet
- **Fingerprint/Face ID:** Login rápido

**Lo que app móvil requeriría:**
- Desarrollo nativo (Swift para iOS, Kotlin para Android)
- Envíos a app store, revisiones, actualizaciones
- Infraestructura de notificaciones push
- Mayor complejidad, más mantenimiento

**Criterios de Decisión:**

| Si... | Entonces... |
|-------|-------------|
| El usuario usa la app principalmente en escritorio | **Solo web** - Suficiente |
| El usuario quiere subida de fotos de recibos | **Construir móvil** - Cámara esencial |
| El usuario necesita acceso offline | **Construir móvil** - Web requiere internet |
| El usuario se siente cómodo con web móvil | **Solo web** - Ahorrar tiempo de desarrollo |

**Recomendación:**
**SOLO WEB (v1).** Medir uso de web móvil. Si > 50% del tráfico es móvil, considerar app nativa.

---

## Pregunta 9: Formatos de Exportación de Datos

**La Pregunta:**
¿Qué formatos de exportación deberían soportarse?

**Actualmente Soportado:**
- CSV (para Excel, Google Sheets)

**Formatos Solicitados:**

| Formato | Caso de Uso | Complejidad |
|---------|-------------|-------------|
| Excel (XLSX) | Reportes formateados con gráficos | Media |
| PDF | Resúmenes mensuales imprimibles | Baja |
| JSON | Integración API, análisis personalizado | Baja |
| QIF (Quicken) | Importar a Quicken desktop | Media |
| OFX (Open Financial Exchange) | Importar a otro software financiero | Alta |

**Criterios de Decisión:**

| Si... | Entonces... |
|-------|-------------|
| El usuario solo necesita datos para hojas de cálculo | **Solo CSV** - Formato universal |
| El usuario quiere imprimir estados mensuales | **Agregar PDF** - Adición simple |
| El usuario quiere acceso API | **Agregar JSON** - Habilita automatización |
| El usuario quiere migrar a otro software | **Agregar OFX** - Formato estándar |

**Recomendación:**
CSV + PDF inicialmente. Agregar otros basándose en solicitudes específicas del usuario.

---

## Pregunta 10: Sincronización de Cuenta vs. Subida Manual

**La Pregunta:**
¿Debería la app soportar sincronización bancaria automática (Plaid, Yodlee)?

**Estado Actual (v1):**
Solo subida manual (usuario descarga PDF del banco, sube a la app)

**Lo que sincronización automática habilitaría:**
- Sin subidas manuales (transacciones aparecen automáticamente)
- Actualizaciones de saldo en tiempo real
- Agregación multi-banco en un lugar

**Lo que sincronización automática requeriría:**
- Integración con Plaid ($0.25 por usuario por mes)
- Revisión de seguridad (almacenar credenciales bancarias)
- Mantenimiento de conexión bancaria (se rompe cuando bancos cambian APIs)
- Lidiar con desafíos MFA, captchas

**Por qué Subida Manual es Mejor (para v1):**
- ✅ **Privacidad:** Sin credenciales bancarias almacenadas en app
- ✅ **Confiabilidad:** Formato PDF raramente cambia
- ✅ **Costo:** Gratis (sin tarifas de Plaid)
- ✅ **Control:** Usuario decide cuándo importar

**Por qué Sincronización Automática es Mejor (para usuarios avanzados):**
- ✅ **Conveniencia:** Configúralo y olvídalo
- ✅ **Tiempo real:** Ver transacciones inmediatamente
- ✅ **Multi-banco:** Agregar 10+ cuentas fácilmente

**Criterios de Decisión:**

| Si... | Entonces... |
|-------|-------------|
| El usuario tiene 1-2 cuentas bancarias | **Subida manual** - Una vez al mes está bien |
| El usuario tiene 10+ cuentas | **Auto-sincronización** - Subida manual demasiado tediosa |
| El usuario sube < 12 veces/año | **Subida manual** - No es carga |
| El usuario quiere chequeos de saldo diarios | **Auto-sincronización** - Manual demasiado lento |

**Recomendación:**
**SUBIDA MANUAL (v1).** Revisitar si el usuario se queja de frecuencia de subida.

---

## Resumen: Marco de Decisión

**Para cada pregunta abierta, aplicar este marco:**

1. **Construir v1 sin la característica**
2. **Instrumentar uso:** Rastrear qué tan seguido el usuario encuentra la limitación
3. **Establecer umbral:** Si punto de dolor ocurre > X veces/mes → Agregar característica
4. **Validar con usuario:** Preguntar "¿Característica Y resolvería problema Z?"
5. **Implementar si sí, diferir si no**

**Evitar:**
- ❌ Construir características "por si acaso"
- ❌ Especular sobre necesidades futuras
- ❌ Agregar complejidad sin valor probado

**Adoptar:**
- ✅ Comenzar simple
- ✅ Aprender del uso real
- ✅ Agregar características cuando emergen puntos de dolor

**Principio Clave:**
Cada característica agregada hace la app ligeramente más compleja. La complejidad tiene un costo. Solo agregar características cuando el beneficio claramente supera el costo.
