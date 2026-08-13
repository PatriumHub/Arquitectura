# 08 — Estado actual vs objetivo

## Resumen ejecutivo

PatriumHub está **operativo** (MVP Fases 0–5 + módulos post-MVP).  
Stack: app PHP monolítica + BD MySQL (`patriumhub.sql` 0.7.0) en Apache/phpMyAdmin, con integraciones WC/MP desde pantallas.

| Tema | Hoy | Siguiente |
|------|-----|-----------|
| Código app | MVP + presupuestos, estados/proyección, edición movimientos, UX móvil | Uso diario + pulido |
| BD | Schema **0.8.0** instalable en un solo SQL | Backups periódicos |
| Docs | Arquitectura + Guía de uso alineadas a v1 actual | Mantener vivo |
| Deploy | Carpeta `/patrium`, rutas `index.php?r=/...` | HTTPS en producción |
| Mercado Pago | Multi-cuenta desde UI → Cuentas | Mantener sync estable |
| WooCommerce | Sync stock/ventas desde Integraciones | Tienda(s) productivas |
| Patrimonio | Personal / consolidado / entidad + snapshots | Rutina de captura |
| Empresas | Resumen, caja, valuación, **Estados y proyección** | Proyecciones al día |

## Diagrama (estado real)

```mermaid
flowchart LR
  App[PatriumHub PHP]
  DB[(patriumhub 0.7.0)]
  UIInt[Integraciones WC/MP]
  Cron[cron sync + snapshots]
  App --> DB
  UIInt --> App
  Cron --> App
```

## Entregado post plan original (producto)

Además de Fases 0–5:

- Presupuestos mensuales (`budget_templates` / `budget_items`) con impacto en pasivos.
- **Estados y proyección** por empresa (plantillas servicios / productos).
- Usuarios con permisos por entidad en Configuración.
- Movimientos editables con recálculo de saldos; borrado de cuentas en saldo 0.
- Nav: Movimientos junto a Presupuestos; Configuración en perfil; MP en Cuentas.
- Menú móvil (hamburger), ocultar cifras, exports CSV, snapshots por entidad.

## Prioridad restante

1. Uso real + backups documentados en operación.  
2. HTTPS / endurecimiento de servidor.  
3. Roadmap de producto (no bloquea el MVP).
