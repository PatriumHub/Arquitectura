# 03 — Arquitectura de sistema

## Vista de componentes

```mermaid
flowchart TB
  subgraph client [Navegador]
    UI[UI PHP + JS]
  end

  subgraph apache [Apache / PatriumHub]
    Front[Vistas / Dashboards]
    API[API interna JSON]
    Auth[Sesion PHP]
    SyncSvc[Servicios Sync]
    WealthSvc[Motor patrimonio]
    Cron[cron/sync.php]
  end

  subgraph data [MySQL]
    DB[(patriumhub)]
  end

  subgraph external [Externos]
    MP1[MP cuenta personal]
    MP2[MP HomeSpot]
    MPn[MP otras]
    WC1[WooCommerce tienda]
    WCn[Otras tiendas]
  end

  UI --> Front
  UI --> API
  Front --> Auth
  API --> Auth
  Auth --> DB
  Front --> WealthSvc
  API --> WealthSvc
  WealthSvc --> DB
  API --> SyncSvc
  Cron --> SyncSvc
  SyncSvc --> DB
  SyncSvc -->|tokens cifrados en BD| MP1
  SyncSvc --> MP2
  SyncSvc --> MPn
  SyncSvc --> WC1
  SyncSvc --> WCn
```

## Deploy lógico (Apache)

| Componente | Ubicación |
|------------|-----------|
| Carpeta web | `/var/www/html/patrium` (= código de la app) |
| Entry point | `patrium/index.php?r=/ruta` |
| MySQL | import via phpMyAdmin |
| Cron | `php /var/www/html/patrium/cron/sync.php` |

URL: `http://IP/patrium/` — ver [guía de deploy](guia-deploy.md).  
Login seed: `admin@patriumhub.local` / `admin123`.

---

## Flujo A — Alta de empresa y patrimonio base

```mermaid
sequenceDiagram
  participant U as Usuario
  participant App as PatriumHub
  participant DB as patriumhub

  U->>App: Crear entidad Empresa
  App->>DB: INSERT entities + companies
  U->>App: Definir ownerships
  App->>DB: INSERT ownerships
  U->>App: Crear cuentas / activos / pasivos
  App->>DB: INSERT accounts, assets, liabilities...
  U->>App: Ver dashboard empresa
  App->>DB: Lee elementos de la entidad
  App-->>U: Patrimonio calculado (sin duplicar)
```

## Flujo B — Conectar Mercado Pago (sin .env)

```mermaid
sequenceDiagram
  participant U as Usuario
  participant UI as Pantalla Integraciones MP
  participant App as PatriumHub
  participant DB as patriumhub
  participant MP as API Mercado Pago

  U->>UI: Nueva cuenta MP
  U->>UI: Nombre, entidad, Access Token, etc.
  UI->>App: Guardar conexión
  App->>App: Cifra token con storage/app.key
  App->>DB: INSERT integrations + credentials cifradas
  U->>UI: Probar conexión / Sync ahora
  App->>DB: Lee y descifra token
  App->>MP: GET saldo / movimientos
  MP-->>App: Datos
  App->>DB: UPSERT accounts / balances / transactions
  App->>DB: INSERT sync_runs
  App-->>U: Saldo y última sync
```

## Flujo C — Conectar WooCommerce (sin .env)

```mermaid
sequenceDiagram
  participant U as Usuario
  participant UI as Pantalla Integraciones WC
  participant App as PatriumHub
  participant DB as patriumhub
  participant WC as WooCommerce REST API

  U->>UI: Nueva tienda
  U->>UI: URL, Consumer Key, Consumer Secret, entidad
  UI->>App: Guardar conexión
  App->>App: Cifra keys con storage/app.key
  App->>DB: INSERT integrations
  U->>UI: Sync productos / stock / pedidos
  App->>WC: REST read-only
  WC-->>App: Catálogo, stock, orders
  App->>DB: inventories + snapshots + métricas ventas
  App-->>U: Dashboard empresa actualizado
```

## Flujo D — Cálculo de patrimonio

```mermaid
flowchart TD
  A[Seleccionar contexto] --> B{Tipo de vista}
  B -->|Entidad| C[Solo elementos de esa entidad]
  B -->|Personal| D[Personales + valor de ownerships]
  B -->|Consolidado| E[Multi-entidad sin dobles]
  C --> F[Sumar cuentas + activos + cobrables + inventario + props]
  D --> F2[Sumar personales + participaciones]
  E --> F3[Sumar selección - internos cruzados]
  F --> G[Restar pasivos]
  F2 --> G
  F3 --> G
  G --> H[Patrimonio neto + composición]
```

**Fórmula conceptual:**

```
Patrimonio = cuentas + activos + propiedades + cuentas por cobrar
           + inventario + participaciones - pasivos
```

Regla anti-duplicación: en patrimonio personal se suma el valor de la **participación**, no los activos internos de la empresa.

---

## Límites de confianza

| Frontera | Regla |
|----------|-------|
| UI → API interna | Sesión autenticada; CSRF en mutaciones |
| App → MySQL | Un solo usuario de app con grants sobre `patriumhub` |
| App → Mercado Pago | Token por conexión, leído de BD, cifrado en reposo |
| App → WooCommerce | Key/secret por tienda, leídos de BD, cifrados en reposo |
| Integraciones → externos | Solo lectura por defecto |
| Cron → sync | Mismo código de servicios; no bypass de cifrado |

## Procesos programados

| Job | Frecuencia sugerida | Acción |
|-----|---------------------|--------|
| Sync MP | cada 15–60 min | Saldos y movimientos de conexiones activas |
| Sync WC | cada 30–120 min | Stock, productos, pedidos/ventas |
| Snapshots patrimonio | diario | Histórico para evolución |
| Limpieza logs | semanal | Retener sync_runs / audit según política |
