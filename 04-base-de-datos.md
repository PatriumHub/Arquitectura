# 04 — Base de datos

## Decisión

**Una sola base de datos** para todo PatriumHub, versionada como un único archivo de instalación:

| Nombre BD | Archivo SQL | Ownership |
|-----------|-------------|-----------|
| `patriumhub` | `databases/patriumhub.sql` | App PatriumHub |

Schema version actual: **0.8.6** (ver `settings.schema.version`).

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
| `receivables` | Dinero a cobrar |
| `liabilities` | Pasivos / obligaciones |
| `currencies` | Catálogo ARS/USD/EUR |
| `tags` / `entity_tags` | Etiquetas opcionales (reservado) |

### Presupuestos y proyección

| Tabla | Uso |
|-------|-----|
| `budget_templates` | Gasto fijo recurrente (mensual) |
| `budget_items` | Instancia del mes (`pending` / `paid` / `skipped`). Solo `pending` con `period_ym <= mes actual` suman a pasivos |
| `company_financial_plans` | Estados y proyección por empresa (`workbook_json` v2 + % ahorro) |
| `person_financial_plans` | Proyección personal (`workbook_json` v2 + % ahorro). Disponible neto = balance − ahorro (meta % solo sobre saldo positivo). UI: promedio mensual = neto ÷ 12; gasto diario = neto del mes ÷ días; carga = egresos + ahorro + disponible neto (y por categoría en ficha) |
| `company_clients` | Fichas de cliente (empresas `services`): contacto, estado, notas; montos sync desde proyección |

Servicios de app: `BudgetService`, `FinancialPlanService` (ahorro solo si balance del mes > 0), `PersonFinancialPlanService`, `ProjectionsAggregateService` (menú `/proyecciones`: flujo, ahorro, carga egresos+ahorro+disponible neto; sin doughnut de composición en el flujo). En ficha persona, `person-financial-plan.js` añade desglose por nombre de egreso.

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
└── patriumhub.sql                 # único SQL de instalación (0.8.6)
```

Instalación nueva: importar ese archivo.  
BD ya en **0.8.5**: la app crea `account_owners` sola al usar Cuentas (`AccountOwnerService::ensureSchema`). Para alinear el número:

```sql
UPDATE settings SET setting_value = '0.8.6', updated_at = CURRENT_TIMESTAMP
WHERE setting_key = 'schema.version';
```

## Qué no vive en esta BD

- Tokens en texto plano.
- Credenciales hardcodeadas de MP/WC.
- Segunda BD por módulo.
- Binarios de contratos/docs (solo metadatos en `documents`).
