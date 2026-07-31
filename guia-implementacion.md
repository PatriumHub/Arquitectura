# 09 — Guía de implementación

Guía operativa para levantar PatriumHub en **local / XAMPP** con Apache + MySQL/MariaDB + phpMyAdmin.

- Deploy en **producción** (HTTPS, cron, backups, updates): **[11 — Guía de deploy](guia-deploy.md)**  
- Detalle corto de BD: [`../databases/apply-phpmyadmin.md`](../databases/apply-phpmyadmin.md)  
- Índice de docs: [README](README.md)

---

## 1. Requisitos

| Componente | Mínimo |
|------------|--------|
| PHP | 8.1+ (`pdo_mysql`, `openssl`, `mbstring`, `json`, `curl`) |
| Apache | con `mod_rewrite` |
| MySQL / MariaDB | 8.0+ / 10.4+ |
| phpMyAdmin | para importar `databases/patriumhub.sql` |

---

## 2. Repos y rutas

```
PatriumHub/                 # monorepo local
├── PatriumHub/             # código (DocumentRoot → public/)
├── databases/              # patriumhub.sql + seeds/ + patches
└── Arquitectura/           # docs
```

En el servidor:

```
/var/www/PatriumHub/public   ← DocumentRoot
```

XAMPP local típico:

```
http://localhost/PatriumHub/public/
```

(si el código vive en `htdocs/PatriumHub` o equivalente vía alias/junction).

---

## 3. Base de datos (phpMyAdmin)

1. Importar `databases/patriumhub.sql` (crea BD + tablas + seeds).
2. Crear usuario MySQL dedicado:

```sql
CREATE USER IF NOT EXISTS 'patrium_app'@'localhost' IDENTIFIED BY 'CAMBIAR_PASSWORD';
GRANT SELECT, INSERT, UPDATE, DELETE ON patriumhub.* TO 'patrium_app'@'localhost';
FLUSH PRIVILEGES;
```

3. Verificación:

```sql
USE patriumhub;
SHOW TABLES;
SELECT email, role FROM users;
```

Usuario seed de la app:

| Campo | Valor |
|-------|-------|
| Email | `admin@patriumhub.local` |
| Password | `admin123` |

No pegar tokens de Mercado Pago ni keys de WooCommerce en el SQL.

---

## 4. Config local de la app

```bash
cd PatriumHub
cp .env.example .env
```

```
DB_HOST=127.0.0.1
DB_NAME=patriumhub
DB_USER=patrium_app
DB_PASS=********
APP_URL=http://localhost/PatriumHub/public
APP_DEBUG=true
```

- En `.env` **solo** MySQL + URL. Sin tokens MP/WC.
- Entrás al sistema → **Configuración** → Generar clave de cifrado (`storage/app.key`).
- Después → **Integraciones** → + WooCommerce / + Mercado Pago.

---

## 5. Apache vhost (ejemplo)

```apache
<VirtualHost *:80>
    ServerName patriumhub.local
    DocumentRoot /var/www/PatriumHub/public

    <Directory /var/www/PatriumHub/public>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/patriumhub-error.log
    CustomLog ${APACHE_LOG_DIR}/patriumhub-access.log combined
</VirtualHost>
```

> **Producción:** checklist HTTPS, vhosts TLS, permisos y hardening → [guía de deploy §2 / §7 / §11](guia-deploy.md).

---

## 6. Cron de sincronización y snapshots

En local (rutas XAMPP según tu install):

```cron
*/30 * * * * php C:/xampp/htdocs/PatriumHub/PatriumHub/cron/sync.php
15 3 * * * php C:/xampp/htdocs/PatriumHub/PatriumHub/cron/snapshots.php
```

En servidor Linux, ver [guía de deploy §9](guia-deploy.md).

- `sync.php`: WooCommerce y Mercado Pago (`sync_auto=true`). Requiere `storage/app.key`.
- `snapshots.php`: historial patrimonial (personal, consolidado, por entidad).

Patches opcionales si la BD es antigua:

- `databases/patch_fase4.sql` (snapshots / saved_views)
- `databases/patch_fase5.sql` (marca schema 0.5.0)

Seed demo opcional (sin tokens): `databases/seeds/demo_minimo.sql` — **no usar en producción real**.

---

## 7. Smoke test

- [ ] Import SQL OK (`users`, `integrations`, etc.)
- [ ] `.env` solo con `DB_*` + `APP_URL` (sin tokens)
- [ ] Login seed + **Configuración** → generar clave
- [ ] Integraciones → WooCommerce (keys en pantalla) → Probar / Sync
- [ ] Integraciones → Mercado Pago (token en pantalla) → Probar / Sync
- [ ] `php cron/sync.php` corre sin error
- [ ] Participaciones + valuación; Dashboard modo Personal ≠ Consolidado con empresas
- [ ] `php cron/snapshots.php` crea filas en `net_worth_snapshots`
- [ ] Toggle **Ocultar cifras** (topbar / Configuración)
- [ ] Export CSV en Cuentas y Movimientos
- [ ] (Opcional) import `databases/seeds/demo_minimo.sql`

---

## 8. Operación de integraciones (Fases 2+)

1. Ir a **Integraciones**.  
2. Elegir proveedor.  
3. Completar entidad + credenciales.  
4. **Probar conexión**.  
5. **Sincronizar ahora** o esperar cron.  

---

## 9. Backup y restore

Procedimiento completo (script, retención, rollback): **[guía de deploy §10](guia-deploy.md)**.

Resumen local:

```bash
mysqldump -u root -p --single-transaction patriumhub > backup-patriumhub.sql
# Guardar también storage/app.key aparte del dump
```

Si se pierde `app.key`, hay que **re-cargar** tokens desde Integraciones.
