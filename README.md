# Arquitectura PatriumHub

Documentación de referencia del sistema **PatriumHub**: centro de comando patrimonial personal y multiempresa.

## Índice

| # | Documento | Qué cubre |
|---|-----------|-----------|
| 01 | [Visión general](01-vision-general.md) | Propósito del sistema y principios |
| 02 | [Plataforma](02-plataforma.md) | App única, stack y límites |
| 03 | [Arquitectura de sistema](03-arquitectura-sistema.md) | Diagrama de componentes y flujos |
| 04 | [Base de datos](04-base-de-datos.md) | Una sola BD y ownership |
| 05 | [Integraciones](05-integraciones.md) | WooCommerce, Mercado Pago, sync |
| 06 | [UI / UX](06-ui-ux.md) | Navegación, dashboards y pantallas |
| 07 | [Seguridad y autenticación](07-seguridad-auth.md) | Sesiones, cifrado de tokens, auditoría |
| 08 | [Estado actual vs objetivo](08-estado-actual-vs-objetivo.md) | Gap analysis |
| 09 | [Guía de implementación](guia-implementacion.md) | Arranque local / XAMPP + phpMyAdmin |
| 10 | [Plan de trabajo](plan-trabajo.md) | Fases del MVP (0–5) |
| 11 | [Guía de deploy](guia-deploy.md) | Apache Linux + SQL en phpMyAdmin |

## Repos del monorepo local

```
PatriumHub/
├── PatriumHub/      # Código PHP (front + back en una sola app)
├── databases/       # patriumhub.sql (instalación única, schema 0.8.6)
├── Guia_De_Uso/     # Manual de usuario en español
└── Arquitectura/    # Esta documentación
```

## Por dónde empezar

| Objetivo | Documento |
|----------|-----------|
| Entender el producto | [01 — Visión](01-vision-general.md) |
| Levantar en local | [09 — Implementación](guia-implementacion.md) |
| Subir a Apache Linux | [11 — Deploy](guia-deploy.md) |
| Ver fases hechas | [10 — Plan de trabajo](plan-trabajo.md) |

## Reglas de oro

1. **Una sola aplicación PHP** en `PatriumHub/` — front y back juntos, modular.
2. **Una sola base de datos** (`patriumhub`) versionada en `databases/`.
3. **Deploy en Apache** + MySQL/MariaDB; carpeta `/patrium` con rutas `index.php?r=/...` (sin tocar Apache).
4. **Credenciales de integraciones en pantallas del sistema**, no en `.env`:
   - cada cuenta de Mercado Pago se configura en UI;
   - cada tienda WooCommerce se configura en UI;
   - los tokens se guardan cifrados en la BD.
5. **Patrimonio, no contabilidad** — calcula valor y explica cambios; no genera libros fiscales.

## Estado de este repo

Documentación viva. **Fases 0–5 aplicadas (MVP completo)** + módulos post-MVP (presupuestos, proyecciones personales/consolidadas, estados de empresa, clientes, métricas de listados, UX operativa).

- Operación local: [guía de implementación](guia-implementacion.md)  
- Operación producción: [guía de deploy](guia-deploy.md)  
- Manual de usuario: [`../Guia_De_Uso/`](../Guia_De_Uso/README.md)  
- BD: un solo archivo [`../databases/patriumhub.sql`](../databases/patriumhub.sql) (schema **0.8.6**)
