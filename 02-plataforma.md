# 02 — Plataforma

## Resumen

| Elemento | Decisión |
|----------|----------|
| App | Una sola: `PatriumHub/` |
| Audiencia | Usuario dueño del patrimonio (uso personal / familiar / multiempresa) |
| Stack | PHP 8+ (front + back), HTML/CSS/JS, MySQL/MariaDB |
| Servidor | Apache |
| BD | Una sola: `patriumhub` en repo `databases/` |
| Deploy BD | phpMyAdmin (import del `.sql`) |
| Framework | PHP modular (MVC ligero). Sin Laravel obligatorio en MVP |
| Contenedores | No requeridos. Apache + MySQL nativos |

---

## Rol de la aplicación

PatriumHub concentra en una sola app:

- Autenticación y usuarios.
- ABM de entidades (personas / empresas).
- Cuentas, activos, propiedades, cobrables, pasivos, inventario, participaciones.
- Movimientos e historial.
- Dashboards fijos con filtros.
- Pantallas de configuración de integraciones (MP / WooCommerce).
- Jobs de sincronización (cron).

### Lo que puede hacer
- Calcular patrimonio por entidad, personal y consolidado.
- Sincronizar datos de lectura desde WooCommerce y Mercado Pago.
- Guardar historial de saldos, valuaciones y syncs.
- Auditar altas, bajas, ediciones y sincronizaciones.

### Lo que no debe hacer
- Generar contabilidad formal / libros fiscales.
- Reemplazar el admin de WooCommerce o el panel de Mercado Pago.
- Escribir en tiendas WC o cuentas MP por defecto (solo lectura).
- Depender de Redis, colas distribuidas o microservicios en el MVP.

---

## Estructura de carpetas propuesta (código)

```
PatriumHub/
├── public/                 # DocumentRoot Apache
│   ├── index.php
│   ├── assets/
│   └── webhooks/           # endpoints públicos (MP, etc. si aplica)
├── app/
│   ├── Controllers/
│   ├── Models/
│   ├── Services/
│   │   ├── Patrimonio/
│   │   ├── Integrations/
│   │   │   ├── MercadoPago/
│   │   │   └── WooCommerce/
│   │   └── Sync/
│   ├── Middleware/
│   └── Views/              # o templates PHP
├── config/
│   ├── app.php
│   └── database.php        # host/user/pass/db (infra local)
├── cron/
│   └── sync.php
├── storage/
│   ├── logs/
│   └── uploads/
└── README.md
```

## Módulos de dominio

| Módulo | Responsabilidad |
|--------|-----------------|
| Identity | Usuarios, roles, sesiones, entidades |
| Wealth | Cuentas, activos, pasivos, cobrables, patrimonio |
| Business | Empresas, inventario, ventas, participaciones |
| Transactions | Movimientos y categorías |
| Integrations | Conexiones WC/MP, pantallas de credenciales, sync |
| Analytics | Métricas, snapshots, dashboards |
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
| Host / user / pass / nombre BD | `config/database.php` o `.env` local de infra | Solo conexión al servidor |
| `APP_KEY` (cifrado) | config local (no en git) | Para cifrar tokens en BD |
| Access token MP por cuenta | **Pantalla Integraciones → Mercado Pago** → BD cifrada | Multi-cuenta, por entidad |
| Consumer key/secret WC por tienda | **Pantalla Integraciones → WooCommerce** → BD cifrada | Multi-tienda, por entidad |
| URL tienda WC, webhook secrets | Misma pantalla de integración | Operable sin redeploy |
