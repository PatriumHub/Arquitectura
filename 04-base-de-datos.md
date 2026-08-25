# 04 — Base de datos

## Decisión

**Una sola base de datos** para todo PatriumHub, versionada como un único archivo de instalación:

| Nombre BD | Archivo SQL | Ownership |
|-----------|-------------|-----------|
| `patriumhub` | `databases/patriumhub.sql` | App PatriumHub |

Schema version actual: **0.8.8** (ver `settings.schema.version`).

> Deploy: importar **solo** `databases/patriumhub.sql` en phpMyAdmin.  
> Los patches históricos quedan absorbidos; no hace falta aplicar varios `.sql`.

## Principios

1. **Charset** `utf8mb4` / collation `utf8mb4_unicode_ci`.
2. **Sin secretos en el SQL** (tokens van cifrados en runtime; seed sin credenciales reales).
3. **Soft-delete o estados** donde aporte trazabilidad (cuentas cerradas, deudas canceladas).
4. **Historial temporal** separado del valor actual (`account_balances`, `inventory_snapshots`, `business_valuations`, `net_worth_snapshots`).
5. **Integraciones como entidades de primer nivel** — no hardcode en config de deploy.

## Diagrama ER conceptual

```mermaid
erDiagram
  USERS ||--o{ AUDIT_LOG : genera
  ENTITIES ||--o| PEOPLE : especializa
  ENTITIES ||--o| COMPANIES : especializa
  ENTITIES ||--o{ ACCOUNTS : posee
  ENTITIES ||--o{ ASSETS : posee
  ENTITIES ||--o{ PROPERTIES : posee
  ENTITIES ||--o{ RECEIVABLES : acredora
  ENTITIES ||--o{ LIABILITIES : debe
  ENTITIES ||--o{ INVENTORIES : stock
  ENTITIES ||--o{ INTEGRATIONS : conecta
  ENTITIES ||--o| COMPANY_FINANCIAL_PLANS : proyecta
  ENTITIES ||--o| PERSON_FINANCIAL_PLANS : proyecta_persona
  ENTITIES ||--o{ COMPANY_CLIENTS : fichas_cliente
  FINANCIAL_GOALS }o--|| CURRENCIES : moneda
  COMPANY_CLIENTS ||--o{ DOCUMENTS : adjuntos
  PEOPLE ||--o{ OWNERSHIPS : participa
  COMPANIES ||--o{ OWNERSHIPS : es_participada
  COMPANIES ||--o{ BUSINESS_VALUATIONS : valuada
  ACCOUNTS ||--o{ ACCOUNT_BALANCES : historial
  ACCOUNTS ||--o{ ACCOUNT_OWNERS : co_titulares
  ENTITIES ||--o{ ACCOUNT_OWNERS : participa_cuenta
  ASSETS ||--o{ ASSET_OWNERS : co_titulares
  ENTITIES ||--o{ ASSET_OWNERS : participa_activo
  ACCOUNTS ||--o{ TRANSACTIONS : mueve
  ENTITIES ||--o{ BUDGET_TEMPLATES : gasta
  BUDGET_TEMPLATES ||--o{ BUDGET_ITEMS : instancia
  INVENTORIES ||--o{ INVENTORY_SNAPSHOTS : historial
  INTEGRATIONS ||--o{ SYNC_RUNS : ejecuta
  USERS ||--o{ SAVED_VIEWS : guarda
```

## Entidades actuales

### Identity / núcleo

| Tabla | Uso |
|-------|-----|
| `users` | Login local (`admin` \| `viewer`) |
| `user_entity_access` | Personas/empresas visibles para usuarios no-admin |
| `entities` | Dueño lógico: `type` = `person` \| `company` |
| `people` | Datos específicos de persona |
| `companies` | Datos de empresa + `business_model` (`services` \| `products`; solo al crear) |
| `ownerships` | % participación persona → empresa |

### Wealth

| Tabla | Uso |
|-------|-----|
| `accounts` | Bancos, billeteras, efectivo, MP, brokers |
| `account_owners` | Co-titulares personas (`ownership_pct` por persona); empresas usan solo `accounts.entity_id` |
| `account_balances` | Historial de saldos |
| `assets` | Activos varios; empresas al 100% en `entity_id` |
| `asset_owners` | Co-titulares personas (`ownership_pct` por persona) |
| `properties` | Inmuebles |
| `receivables` | Dinero a cobrar (`due_date`, status incl. `paid`). UI: cabeceras ordenables; ficha entidad muestra Vence; alta/cobro desde ficha vuelve con `return_to` (`safe_return_path`) |
| `liabilities` | Pasivos / obligaciones |
| `currencies` | Catálogo ARS/USD/EUR |
| `tags` / `entity_tags` | Etiquetas opcionales (reservado) |

### Presupuestos y proyección

| Tabla | Uso |
|-------|-----|
| `budget_templates` | Gasto fijo recurrente (mensual); `category_id` opcional → `transaction_categories` |
| `budget_items` | Instancia del mes (`pending` / `paid` / `skipped`). Solo `pending` con `period_ym <= mes actual` suman a pasivos. Copia `category_id` de la plantilla; al pagar se asigna al egreso |
| `company_financial_plans` | Estados y proyección por empresa (`workbook_json` v2 + % ahorro). Misma regla de disponible neto que persona. UI: año default = calendario; Comparativa ↔ Detalle sincronizados; carga de egresos (anillo + mes a mes + por categoría) y gasto diario máximo (neto ÷ días; ref. ÷ 30); `+ Año` sin prompt |
| `person_financial_plans` | Proyección personal (`workbook_json` v2 + % ahorro). Disponible neto = balance − ahorro (meta % solo sobre saldo positivo). UI: año default = calendario; promedio mensual = neto ÷ 12; gasto diario = neto del mes ÷ días; carga = egresos + ahorro + disponible neto (y por categoría en ficha) |
| `financial_goals` | Metas personalizadas (nombre, meta, juntado, fechas, moneda). Milestones NO son filas: MS01/MS03 usan `settings`; MS02 lee `liabilities` de personas |
| `company_clients` | Fichas de cliente (empresas `services`): contacto, estado, notas; montos sync desde proyección |

Servicios de app: `BudgetService`, `FinancialPlanService`, `PersonFinancialPlanService`, `ProjectionsAggregateService`, `FinancialGoalsService` (`/objetivos`: Cumplidos N/N, MS03 por año, alta en `/objetivos/nuevo`). En ficha persona, `person-financial-plan.js` añade desglose por nombre de egreso.

### Business / inventario

| Tabla | Uso |
|-------|-----|
| `inventories` | Resumen de stock por entidad/fuente |
| `inventory_snapshots` | Historial valor/unidades |
| `business_valuations` | Valuación de empresa + método |
| `sales_metrics` | Agregados WC por período |

### Movimientos

| Tabla | Uso |
|-------|-----|
| `transactions` | Ingresos, egresos, transferencias, ajustes |
| `transaction_categories` | Categorías (Sueldos, Honorarios, WC, etc.) |

### Integraciones

| Tabla | Uso |
|-------|-----|
| `integrations` | Conexión: provider, entity_id, nombre, estado |
| `integration_credentials` | Keys/tokens **cifrados** |
| `sync_runs` | Ejecuciones, OK/error, conteos |
| `sync_cursors` | Paginación / last_id / last_date |

### Historial y soporte

| Tabla | Uso |
|-------|-----|
| `net_worth_snapshots` | Snapshots personal / consolidado / entidad |
| `saved_views` | Filtros de dashboard guardados |
| `documents` | Adjuntos; hoy: contratos/docs de `company_clients` |
| `audit_log` | Quién cambió qué |
| `settings` | Preferencias de app |

## `company_financial_plans` (Estados y proyección)

| Campo | Nota |
|-------|------|
| `entity_id` | FK única a empresa |
| `workbook_json` | Libro v2 + `template` `services`\|`products` |
| Servicios | `clients[]` × mes + `expenses[]` |
| Productos | `income_months[12]` + `expenses[]` (sin clientes) |
| Creación | Al alta según `companies.business_model` |
| Seed Soup IT | Solo servicios; Excel en `storage/templates/` |
| UI al pie | Carga de egresos + gasto diario máximo (misma lógica que persona) |
| Año activo | Default = calendario; Comparativa y Detalle sincronizados; `+ Año` = siguiente año |

## `company_clients` (pestaña Clientes)

| Campo / regla | Nota |
|---------------|------|
| Alcance | Solo empresas con `business_model = services` |
| Lista / montos | Fuente de verdad: filas de cliente en `workbook_json` (sync al abrir Clientes o guardar proyección) |
| Ficha | `contact_name`, `email`, `phone`, `contract_notes`, `is_active` |
| `value_annual` / `value_total` | Recalculados desde la proyección |
| Archivos | Via `documents`: `company_client_contract` (PDF) y `company_client_file`; disco en `storage/uploads/client_docs/` |

## Repo `databases/`

```
databases/
└── patriumhub.sql                 # único SQL de instalación (0.8.8)
```

Instalación nueva: importar ese archivo.  
BD ya en **0.8.7** o anterior con tablas al día: solo alinear el número si hace falta:

```sql
UPDATE settings SET setting_value = '0.8.8', updated_at = CURRENT_TIMESTAMP
WHERE setting_key = 'schema.version';
```

## Qué no vive en esta BD

- Tokens en texto plano.
- Credenciales hardcodeadas de MP/WC.
- Segunda BD por módulo.
- Binarios de contratos/docs (solo metadatos en `documents`).
- Preferencia de tema claro/oscuro (`localStorage patrium-theme` en el navegador).
- Estado del botón “volver arriba” (solo UI).
