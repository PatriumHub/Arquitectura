# 07 — Seguridad y autenticación

## Modelo de auth (MVP)

```mermaid
flowchart TB
  Login[Login email/password] --> Session[Sesion PHP server-side]
  Session --> UI[UI protegida]
  Session --> API[API interna]
  API --> Crypto[Descifrado credentials]
  Crypto --> Ext[APIs externas]
```

- Autenticación por **sesión PHP** contra `users`.
- Cookie de sesión e idle timeout: **24 h** por defecto (`SESSION_LIFETIME` / `SESSION_IDLE` = `86400`). Cada visita autenticada renueva `_last_activity` y reenvía la cookie (ventana deslizante): si el usuario vuelve antes de las 24 h, sigue adentro; si pasa ese lapso sin entrar, al próximo request se cierra. `session.gc_maxlifetime` se alinea con el lifetime para que el GC del server no borre el archivo antes.
- MFA: previsto a futuro, no bloqueante del MVP.
- Roles mínimos: `admin` (todo) y, si hace falta, `viewer` (solo lectura).
- Permisos por entidad: roadmap (compartir acceso familiar); MVP asume un operador principal.

## Credenciales de integraciones

| Regla | Detalle |
|-------|---------|
| Almacenamiento | Solo en BD, columna cifrada |
| Algoritmo | AES-256-GCM con clave de `storage/app.key` (Configuración) |
| Visualización | La UI **no** re-muestra el token completo tras guardar; solo “••••” + rotar |
| Revocación | Independiente por conexión |
| Alcance | Mostrar scopes / permisos conocidos |
| Tránsito | HTTPS obligatorio en producción |
| Logs | Nunca loguear tokens en claro |

## Controles de aplicación

- CSRF en formularios y mutaciones JSON.
- Password hashing (`password_hash` / Argon2id o bcrypt).
- Rate limit básico en login.
- Prepared statements en todo SQL.
- Uploads validados (tipo/tamaño) en `documents`.
- `.htaccess` bloquea `app/`, `config/`, `storage/`, `cron/`, `.env`.

## Auditoría y privacidad

- `audit_log` para altas, bajas, ediciones sensibles y cambios de integraciones.
- Cada sync deja rastro en `sync_runs`.
- Posibilidad de ocultar cifras en pantalla.
- Exportación y borrado completo de datos (criterio de producto).
- Backups de MySQL fuera del DocumentRoot.

## Secretos: mapa final

| Secreto | Dónde | En git |
|---------|-------|--------|
| DB credentials | `.env` infra | No |
| Clave cifrado | `storage/app.key` (Configuración) | No |
| Tokens MP (N cuentas) | BD cifrada vía menú Integraciones | No |
| Keys WooCommerce (N tiendas) | BD cifrada vía menú Integraciones | No |
| Schema SQL | `databases/patriumhub.sql` | Sí |
| Seeds demo | sin secretos reales | Sí |

## Superficie pública

| Endpoint | Auth |
|----------|------|
| `index.php?r=/login` | Público |
| Resto de UI | Sesión |
| Webhooks futuros | Firma/secret por integración (si se habilitan) |
| Cron CLI | Solo ejecución local/servidor, no HTTP público |

## Deploy

Apache + phpMyAdmin, sin tocar el server: **[11 — Guía de deploy](guia-deploy.md)**.
