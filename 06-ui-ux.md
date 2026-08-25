# 06 — UI / UX

## Una experiencia, un trabajo claro

PatriumHub es una sola app. La UI prioriza **comprensión patrimonial** sobre densidad de un ERP.

Tema visual: claro por defecto; **modo oscuro** con `html[data-theme]` + `localStorage` (`patrium-theme`), mismo patrón que el sitio. Acento verde, tipografía **DM Sans** + **IBM Plex Mono** (cifras). Toggle sol/luna siempre visible en la barra (desktop: a la derecha del nav; móvil: entre brand y hamburger). Al cambiar tema se dispara `patrium:theme` y los Chart.js de listados/gastos/gastos agrupados se **vuelven a pintar** (ticks/leyendas legibles en ambos modos).

Acciones de filas en tablas: helpers `ui_icon` / `icon_action_link` / `icon_action_button` (clase `.btn-icon`). Convención de color: **Eliminar** = rojo (`danger`); **Pagar / Cobrar** = amarillo (`warn`). Las celdas de acción usan `td.actions` como `table-cell` (no `display:flex` del toolbar `.page-head .actions`).

Tokens de contraste (no hay preferencia en BD):

| Superficie | Token | Oscuro |
|------------|-------|--------|
| Cards KPI “hero” (`.stat.accent`) | `--navy` (superficie oscura en ambos temas) | slate `#243044` + texto blanco |
| Cabeceras de tabla | `--table-head` | `#1c2430` |
| Filas | `--table-row` / `--table-row-hover` | `#171d25` / `#1c2430` |
| Cards tintadas (`tone-*`) | pasteles en claro | fondos `rgba(...)` sobre `--bg-elev` |

```mermaid
flowchart LR
  Brand[PatriumHub] --> Dash[Dashboards]
  Brand --> Goals[Objetivos]
  Brand --> Proj[Proyecciones]
  Brand --> Master[ABMs patrimonio]
  Brand --> Budgets[Presupuestos]
  Brand --> Tx[Movimientos]
  Brand --> Int[Integraciones]
  Brand --> Config[Configuracion]
```

---

## Navegación principal

Barra superior (desktop) / panel hamburger (móvil):

1. **Inicio** — resumen rápido (+ fila % sobre activos)  
2. **Dashboard** — gráficos, snapshots, vistas guardadas (+ fila % sobre activos)  
3. **Objetivos** — Objetivos fundamentales (milestones + metas; alta en `/objetivos/nuevo`)  
4. **Proyecciones** — flujo, ahorro, carga de egresos y gasto diario (solo lectura)  
5. **Presupuestos**  
6. **Gastos** — análisis de egresos (real + presupuesto + proyección; `/gastos`)  
7. **Movimientos**  
8. **Patrimonio** ▾ — Personas, Empresas, Participaciones, Cuentas, Activos varios, Propiedades, Cobrables, Pasivos, Inventario (último; summary con accent ámbar)  
9. **Perfil** ▾ — Configuración (perfil / usuarios / sistema), Integraciones, Salir  
10. Toggle **Ocultar cifras**
11. Toggle **sol/luna** (tema claro/oscuro, por navegador)
12. Botón **volver arriba** (esquina inferior derecha, aparece al scrollear)

Configuración: cambiar nombre/email/contraseña; admin crea usuarios y tilda personas/empresas visibles.

> Mercado Pago no tiene ítem de menú propio: vive en **Cuentas** (tipo MP) y en el dashboard de empresa / analytics.

---

## Dashboards

Filtros globales (moneda, entidad, fechas, origen) + chips activos + **vistas guardadas**.

### Dashboard general

| Panel | Tipo |
|-------|------|
| KPIs | Patrimonio neto, activos, pasivos, liquidez |
| % sobre activos | Segunda fila: qué % de los activos totales es cada KPI de arriba |
| Composición | Cuentas, activos, propiedades, cobrables, inventario, participaciones |
| Por entidad | Barras / participación |
| Evolución | Snapshots (neto, activos, pasivos) |
| Flujo | Ingresos vs egresos del período |
| Complemento | Últimos movimientos, capturar snapshot |

### Objetivos (menú)

Ruta `GET /objetivos` · alta `GET/POST /objetivos/nuevo` · progreso/eliminar por id · POST fondo / ahorro-15.

1. **Milestones** (cards full-width + **Cumplidos N/N**):
   - 01 fondo emergencia ARS 1.2M (juntado manual en `settings`)
   - 02 deudas = pasivos abiertos de entidades `person`
   - 03 meta auto = 15% ingreso neto personas del año en curso; ahorrado manual `goals.ramsey.ms03_saved_{año}_{moneda}`
2. Corte visual.
3. **Tus objetivos** (Cumplidos N/N + Agregar): listado `financial_goals`; formulario solo en `/objetivos/nuevo`.

### Proyecciones (menú)

Orden de bloques:

1. KPIs de flujo (incluye **promedio mensual** = disponible neto anual ÷ 12) y tiles de realidad  
2. **Flujo del año** (gráfico ancho ingresos/egresos/balance; sin doughnut de composición) + personas vs empresas + por entidad  
3. **Ahorro proyectado**  
4. Tablas Personas / Empresas  
5. **Carga de egresos** al final:
   - Consolidado: anillo egresos + ahorro + disponible neto; mes a mes apilado; por entidad horizontal; personas vs empresas  
   - Persona (ficha): lo mismo a nivel individual + **por categoría** (nombre de cada línea de egreso × % del ingreso)  
6. **Gasto diario máximo**: 12 cards = disponible neto **del mes** (tras ahorro) ÷ días de ese mes; línea de referencia = promedio mensual ÷ 30; gráfico mes a mes.

Filtros: año, alcance, moneda. Paleta unificada verde/coral. La carga de egresos usa egresos de planilla ÷ ingresos (no `budget_templates`). El verde del bloque es **disponible neto** (balance − ahorro), no el balance bruto. Partial de KPIs %: `app/Views/partials/stat_pct_of_assets.php` (Inicio + Dashboard).

### Ficha empresa — pestañas

`Resumen · Cuentas · [Clientes] · Estados y proyección · Activos varios · Pasivos · Propiedades · Inventario · Cobrables · Movimientos · Integraciones · Participaciones · Valuación`

**Clientes** (solo `business_model = services`): listado sync desde proyección, KPIs activo/inactivo, ficha con contacto, contrato PDF y docs extra.

**Estados y proyección:** dashboard (Comparativa por año + Detalle del año) + tablas HTML +, al final, **carga de egresos** (egresos + ahorro + disponible neto; por nombre de costo) y **gasto diario máximo** (neto tras ahorro ÷ días; ref. ÷ 30).  
- Año activo: por defecto el **año calendario**; el select de Comparativa y las pills de Detalle comparten el mismo índice.  
- **+ Año** agrega el siguiente año numérico (sin prompt); doble clic en la pill para renombrar.  
- Servicios: clientes × mes + costos.  
- Productos: ingresos totales por mes + costos.  
Un libro por empresa; hojas por año. El tipo de negocio se elige **solo al crear** la empresa.

### Ficha persona — pestañas

`Resumen · Proyecciones · Cuentas · Activos · Pasivos · Propiedades · Cobrables · Movimientos · Participaciones`

**Proyecciones:** tablero con neto real, KPIs del año, **promedio mensual** (disponible neto ÷ 12), **flujo del año** a ancho completo, planilla editable y, al final, **carga de egresos** (egresos + ahorro + disponible neto; por nombre de egreso) + **gasto diario máximo** (neto tras ahorro ÷ días; ref. ÷ 30). Año activo por defecto = calendario. Sin gráfico de “composición”.

### Listados de patrimonio

Cada listado (cuentas, activos, propiedades, cobrables, pasivos, inventario, personas, empresas, participaciones) muestra **métricas** arriba y, cuando aplica, gráficos por tipo / entidad + filtro de moneda. **Cobrables:** cabeceras ordenables; en ficha persona/empresa también columna **Vence**; alta/cobro desde ficha vuelve a la entidad (`return_to`).

### Gastos (menú)

Ruta `GET /gastos` · `ExpenseAnalysisService` + `ExpensesController`.

Filtros: `preset` (30d/month/90d/ytd/year/custom), `from`/`to`, `currency`, y `entity_id` con valores:
- vacío → todas las entidades (`scope=all`)
- `people` / `companies` → todas las personas o todas las empresas (`ExpenseAnalysisService` resuelve IDs vía `Catalog::entities`)
- id numérico → una entidad (`scope=entity`)

KPIs y gráficos cruzan:
1. **Real** — `transactions` tipo `expense` / `payment` (/ `egreso`)
2. **Presupuesto** — totales de `budget_items` del mes de cierre del rango (mismo filtro de entidades)
3. **Proyección** — egreso anual de planilla prorrateado al largo del período (`ProjectionsAggregateService` con scope people/companies/all, o planilla de la entidad)

Incluye barras apiladas **horizontales** categoría × mes del año calendario (`by_category_month`: siempre ene–dic del año de `to`; colores; tooltip monto + % del mes). Rankings entidad/categoría/cuenta también en barras horizontales. Categorías de movimiento enriquecidas vía seed + `Catalog::ensureTransactionCategories` (Comida, Salidas, Suscripciones, Alquiler/es, etc.).

### Cuentas

Listado con saldos, gráficos de composición, alta/edición/borrado (saldo 0). Incluye cuentas Mercado Pago sincronizadas. Acciones de fila con iconos (Editar / Movimiento / Eliminar).

### Movimientos

Listado paginado; editar y **eliminar** (`POST /movimientos/{id}/eliminar`) recalculan saldos. Si el movimiento estaba ligado a un `budget_items` pagado, el ítem vuelve a `pending`.

### Activos varios

Alta/edición + **eliminar** (`POST /activos/{id}/eliminar`) desde listado y ficha; borra `asset_owners` y el activo.

---

## Flujos UX críticos

### Alta de empresa
1. Crear empresa eligiendo tipo servicios/productos (se crea plan de Estados y proyección; Soup IT puede seedearse).  
2. Definir ownerships.  
3. Crear cuentas o conectar integraciones.  
4. Cargar activos / pasivos / cobrables.  
5. Completar **Estados y proyección** si aplica.  
6. En servicios: completar fichas en **Clientes** (contacto, contrato, docs).  
7. Revisar resumen / valuación / menú Proyecciones.

### Alta de cuenta
Desde Cuentas: titularidad persona(s) con % o empresa; saldo; tipo. Las cuentas personales pueden compartirse entre varias personas.

### Conectar integración
Perfil → Integraciones → proveedor → entidad + credenciales → probar → sync.

### Presupuesto del mes
Presupuestos → asegurar período → pagar / omitir / revertir.  
Solo los `pending` con `period_ym` ≤ mes actual suman a pasivos. Navegar un mes futuro no baja el neto.  
**Gastos agrupados:** por entidad, con composición (% por nombre) y ranking de mayor a menor; los charts reaccionan al toggle de tema.

### Dinero prestado
Alta como cobrable → marcar pago (acredita en cuenta) hasta cancelar.  
Desde ficha persona/empresa: al guardar o cobrar volvés a esa pestaña **Cobrables**.

---

## Estados de interfaz

| Estado | Tratamiento |
|--------|-------------|
| Vacío | Empty state con CTA |
| Carga | Skeleton en KPIs/tablas densas |
| Error sync | Badge / mensaje en integraciones |
| Privacidad | Clase `hide-amounts` en body |
| Scroll largo | Botón fijo `#back-to-top` (aparece tras ~280px) |
