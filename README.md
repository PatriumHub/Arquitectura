# Arquitectura PatriumHub

Documentación de referencia del sistema **PatriumHub**: centro de comando patrimonial personal y multiempresa.

## Contenido

| Documento | Qué cubre |
|-----------|-----------|
| [01 — Visión general](01-vision-general.md) | Propósito del sistema y principios |
| [02 — Plataforma](02-plataforma.md) | App única, stack y límites |
| [03 — Arquitectura de sistema](03-arquitectura-sistema.md) | Diagrama de componentes y flujos |
| [04 — Base de datos](04-base-de-datos.md) | Una sola BD y ownership |
| [05 — Integraciones](05-integraciones.md) | WooCommerce, Mercado Pago, sync |
| [06 — UI / UX](06-ui-ux.md) | Navegación, dashboards y pantallas |
| [07 — Seguridad y autenticación](07-seguridad-auth.md) | Sesiones, cifrado de tokens, auditoría |
| [08 — Estado actual vs objetivo](08-estado-actual-vs-objetivo.md) | Gap analysis |
| [09 — Guía de implementación](guia-implementacion.md) | Deploy Apache + phpMyAdmin |
| [10 — Plan de trabajo](plan-trabajo.md) | Plan propuesto por fases |

## Repos del monorepo local

```
PatriumHub/
├── PatriumHub/      # Código PHP (front + back en una sola app)
├── databases/       # Schema SQL versionado (una BD)
└── Arquitectura/    # Esta documentación
```

## Reglas de oro

1. **Una sola aplicación PHP** en `PatriumHub/` — front y back juntos, modular.
2. **Una sola base de datos** (`patriumhub`) versionada en `databases/`.
3. **Deploy en Apache** + MySQL/MariaDB vía phpMyAdmin.
4. **Credenciales de integraciones en pantallas del sistema**, no en `.env`:
   - cada cuenta de Mercado Pago se configura en UI;
   - cada tienda WooCommerce se configura en UI;
   - los tokens se guardan cifrados en la BD.
5. **Patrimonio, no contabilidad** — calcula valor y explica cambios; no genera libros fiscales.

## Estado de este repo

Documentación viva. Empezá por el [plan de trabajo](plan-trabajo.md) o la [guía de implementación](guia-implementacion.md).

Hasta validar el plan **no se implementa código ni SQL** en `PatriumHub/` / `databases/`.
