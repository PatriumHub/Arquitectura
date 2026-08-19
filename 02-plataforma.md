# 02 — Plataforma

## Resumen

| Elemento | Decisión |
|----------|----------|
| App | Una sola: `PatriumHub/` |
| Audiencia | Usuario dueño del patrimonio (uso personal / familiar / multiempresa) |
| Stack | PHP 8+ (front + back), HTML/CSS/JS, MySQL/MariaDB |
| Servidor | Apache |
| BD | Una sola: `patriumhub` — install `databases/patriumhub.sql` (0.8.7) |
| Deploy BD | phpMyAdmin (un solo import; sin patches encadenados) |
| Framework | PHP modular (MVC ligero). Sin Laravel obligatorio en MVP |
| Contenedores | No requeridos. Apache + MySQL nativos |

---

## Rol de la aplicación

PatriumHub concentra en una sola app:

- Autenticación y usuarios.
- ABM de entidades (personas / empresas).
- Cuentas, activos, propiedades, cobrables, pasivos, inventario, participaciones.
- Presupuestos mensuales y liquidación (pending del mes actual/atrasados → pasivo).
- Proyección personal (`person_financial_plans`) y Estados y proyección por empresa.
- Vista consolidada **Proyecciones** (`/proyecciones`, solo lectura).
- Fichas de clientes (empresas de servicios): contacto, estado, contrato PDF y docs.
- Listados de patrimonio con métricas y gráficos (totales, por tipo / entidad).
- Movimientos e historial (alta/edición con ajuste de saldos).
- Dashboards fijos con filtros y vistas guardadas.
- Pantallas de configuración de integraciones (MP / WooCommerce).
- Jobs de sincronización y snapshots (cron).

### Lo que puede hacer
- Calcular patrimonio por entidad, personal y consolidado.
- Planificar flujo y ahorro anual (persona, empresa y consolidado) sin tocar cuentas.
- Sincronizar datos de lectura desde WooCommerce y Mercado Pago.
- Guardar historial de saldos, valuaciones, snapshots y syncs.
- Auditar altas, bajas, ediciones y sincronizaciones.

### Lo que no debe hacer
- Generar contabilidad formal / libros fiscales.
- Reemplazar el admin de WooCommerce o el panel de Mercado Pago.
- Escribir en tiendas WC o cuentas MP por defecto (solo lectura).
- Depender de Redis, colas distribuidas o microservicios en el MVP.

---

## Estructura de carpetas propuesta (código)

```
PatriumHub/                 # se copia al server como /patrium
├── index.php               # entry point
├── .htaccess
├── assets/
├── app/
│   ├── Controllers/
│   ├── Services/
│   └── Views/
├── config/
├── cron/
├── storage/
└── README.md
```

## Módulos de dominio

| Módulo | Responsabilidad |
|--------|-----------------|
| Identity | Usuarios, roles, sesiones, entidades |
| Wealth | Cuentas, activos, pasivos, cobrables, patrimonio |
| Planning | Presupuestos, planillas persona/empresa, proyecciones consolidadas |
| Business | Empresas, inventario, ventas, participaciones |
| Transactions | Movimientos y categorías |
| Integrations | Conexiones WC/MP, pantallas de credenciales, sync |
| Analytics | Métricas, snapshots, dashboards, gráficos de listados |
| Documents | Adjuntos |
| Audit | Historial y trazabilidad |

## Relación con repos

| Repo | Contenido |
|------|-----------|
| `PatriumHub/` | Código PHP de la app |
| `databases/` | `patriumhub.sql` (+ seeds demo opcionales) |
| `Arquitectura/` | Esta documentación |

## Configuración: qué va dónde

| Dato | Dónde | Por qué |
|------|-------|---------|
| Host / user / pass / nombre BD | `.env` (solo infra) | Conexión al servidor |
| Clave de cifrado | **Configuración** → `storage/app.key` | Cifra tokens en BD |
| Access token MP por cuenta | **Integraciones → Mercado Pago** → BD cifrada | Multi-cuenta, por entidad |
| Consumer key/secret WC por tienda | **Integraciones → WooCommerce** → BD cifrada | Multi-tienda, por entidad |
