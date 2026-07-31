# 11 — Guía de deploy (producción)

Cómo publicar PatriumHub en un servidor **Apache + MySQL/MariaDB** con HTTPS, cron y backups.

Para el **primer arranque en local / XAMPP**, usá [09 — Guía de implementación](guia-implementacion.md).  
Import corto de BD: [`../databases/apply-phpmyadmin.md`](../databases/apply-phpmyadmin.md).

---

## 1. Alcance

| Escenario | Documento |
|-----------|-----------|
| Dev en notebook (XAMPP) | [guia-implementacion.md](guia-implementacion.md) |
| Producción / VPS / hosting propio | **Este documento** |
| Diseño de seguridad | [07-seguridad-auth.md](07-seguridad-auth.md) |

Stack objetivo: PHP 8.1+, Apache (`mod_rewrite` + TLS), MySQL 8 / MariaDB 10.4+, una sola app en `PatriumHub/public`.

---

## 2. Checklist previo

- [ ] Dominio apuntando al servidor (A/AAAA)
- [ ] Acceso SSH (o panel) al host
- [ ] PHP con extensiones: `pdo_mysql`, `openssl`, `mbstring`, `json`, `curl`
- [ ] Apache con `mod_rewrite`, `mod_ssl`, `AllowOverride All` en el vhost
- [ ] MySQL/MariaDB accesible solo desde localhost (salvo arquitectura deliberada)
- [ ] Certificado TLS (Let’s Encrypt u otro)
- [ ] Directorio de backups **fuera** del DocumentRoot

---

## 3. Layout en el servidor

```
/var/www/PatriumHub/                 # raíz del monorepo (o solo la app)
├── PatriumHub/                      # código PHP
│   ├── public/                      ← DocumentRoot (único path público)
│   ├── app/
│   ├── config/
│   ├── cron/
│   ├── storage/                     # app.key, logs (NO público)
│   └── .env                         # NO versionar / NO público
├── databases/                       # SQL (opcional en el server)
└── Arquitectura/                    # docs (opcional en el server)
```

**Regla:** el DocumentRoot apunta **solo** a `.../PatriumHub/public`.  
Nunca exponer `app/`, `config/`, `storage/`, `.env` ni `databases/`.

Ejemplo de permisos (Linux, usuario del vhost `www-data`):

```bash
sudo chown -R deploy:www-data /var/www/PatriumHub
sudo find /var/www/PatriumHub -type d -exec chmod 755 {} \;
sudo find /var/www/PatriumHub -type f -exec chmod 644 {} \;
sudo chmod 750 /var/www/PatriumHub/PatriumHub/storage
sudo chmod 640 /var/www/PatriumHub/PatriumHub/.env
# tras generar la clave:
sudo chmod 640 /var/www/PatriumHub/PatriumHub/storage/app.key
```

---

## 4. Publicar el código

### Opción A — Git (recomendada)

```bash
cd /var/www
sudo git clone <URL_DEL_REPO> PatriumHub
cd PatriumHub/PatriumHub
cp .env.example .env
# editar .env (sección 6)
```

Actualizaciones:

```bash
cd /var/www/PatriumHub
git pull
# aplicar patches SQL si el release lo indica (sección 12)
```

### Opción B — Copia de artefactos

Subí el árbol `PatriumHub/` (código) + `databases/` por SFTP/rsync.  
Excluí siempre: `.env`, `storage/app.key`, `storage/logs/*`.

```bash
rsync -av --delete \
  --exclude '.env' \
  --exclude 'storage/app.key' \
  --exclude 'storage/logs/' \
  ./PatriumHub/ user@host:/var/www/PatriumHub/PatriumHub/
```

---

## 5. Base de datos

1. Crear BD y usuario de aplicación (mínimos privilegios):

```sql
CREATE DATABASE IF NOT EXISTS patriumhub
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER IF NOT EXISTS 'patrium_app'@'localhost' IDENTIFIED BY 'PASSWORD_FUERTE';
GRANT SELECT, INSERT, UPDATE, DELETE ON patriumhub.* TO 'patrium_app'@'localhost';
FLUSH PRIVILEGES;
```

2. Importar schema:

```bash
mysql -u root -p patriumhub < /var/www/PatriumHub/databases/patriumhub.sql
```

O vía phpMyAdmin → Importar → `patriumhub.sql`.

3. **No** importar `databases/seeds/demo_minimo.sql` en producción real (es escenario de prueba).

4. Verificar:

```sql
USE patriumhub;
SHOW TABLES;
SELECT setting_value FROM settings WHERE setting_key = 'schema.version';
SELECT email, role FROM users;
```

---

## 6. Variables de entorno (producción)

Archivo: `PatriumHub/.env` (basado en `.env.example`).

```env
APP_NAME=PatriumHub
APP_ENV=production
APP_URL=https://patrimonio.tudominio.com
APP_DEBUG=false

DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=patriumhub
DB_USER=patrium_app
DB_PASS=PASSWORD_FUERTE

SESSION_NAME=patriumhub_session
SESSION_LIFETIME=28800
SESSION_IDLE=7200
SESSION_SECURE=true
```

| Clave | Notas |
|-------|--------|
| `APP_URL` | URL pública exacta (HTTPS). Sin slash final inconsistente con redirects. |
| `APP_DEBUG` | Siempre `false` en prod. |
| `SESSION_SECURE` | `true` detrás de HTTPS. En `APP_ENV=production` la app también fuerza cookie Secure. |
| Tokens WC/MP | **Nunca** en `.env` — solo menú Integraciones. |

---

## 7. Apache + HTTPS

### 7.1 Vhost HTTP → HTTPS

```apache
<VirtualHost *:80>
    ServerName patrimonio.tudominio.com
    Redirect permanent / https://patrimonio.tudominio.com/
</VirtualHost>
```

### 7.2 Vhost HTTPS

```apache
<VirtualHost *:443>
    ServerName patrimonio.tudominio.com
    DocumentRoot /var/www/PatriumHub/PatriumHub/public

    <Directory /var/www/PatriumHub/PatriumHub/public>
        AllowOverride All
        Require all granted
        Options -Indexes
    </Directory>

    # Denegar acceso a paths sensibles si el DocumentRoot se configuró mal
    <DirectoryMatch "^/var/www/PatriumHub/PatriumHub/(app|config|storage|cron)">
        Require all denied
    </DirectoryMatch>

    ErrorLog ${APACHE_LOG_DIR}/patriumhub-error.log
    CustomLog ${APACHE_LOG_DIR}/patriumhub-access.log combined

    SSLEngine on
    SSLCertificateFile      /etc/letsencrypt/live/patrimonio.tudominio.com/fullchain.pem
    SSLCertificateKeyFile   /etc/letsencrypt/live/patrimonio.tudominio.com/privkey.pem
</VirtualHost>
```

Let’s Encrypt (ejemplo Certbot):

```bash
sudo certbot --apache -d patrimonio.tudominio.com
sudo apache2ctl configtest && sudo systemctl reload apache2
```

La app ya envía en producción: `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Strict-Transport-Security`.  
`public/.htaccess` enruta requests al front controller.

---

## 8. Primer arranque en producción

1. Abrí `https://patrimonio.tudominio.com/login`
2. Login seed: `admin@patriumhub.local` / `admin123`
3. **Cambiar password del admin de inmediato** (o crear usuario admin nuevo y desactivar el seed)
4. **Configuración** → Generar clave de cifrado → se crea `storage/app.key`
5. Respaldá `storage/app.key` fuera del servidor (sin esa clave no se recuperan tokens WC/MP)
6. **Integraciones** → conectar WooCommerce / Mercado Pago reales
7. Probar conexión + Sync manual
8. Verificar dashboards (general, MP, modos personal/consolidado)

---

## 9. Cron

Usuario del sistema con permiso de lectura sobre el código y la clave:

```cron
# Sync integraciones (WC + MP con sync_auto)
*/30 * * * * /usr/bin/php /var/www/PatriumHub/PatriumHub/cron/sync.php >> /var/log/patriumhub-sync.log 2>&1

# Snapshots patrimoniales diarios
15 3 * * * /usr/bin/php /var/www/PatriumHub/PatriumHub/cron/snapshots.php >> /var/log/patriumhub-snapshots.log 2>&1
```

Prueba manual:

```bash
sudo -u www-data php /var/www/PatriumHub/PatriumHub/cron/sync.php
sudo -u www-data php /var/www/PatriumHub/PatriumHub/cron/snapshots.php
```

`sync.php` falla si no existe `storage/app.key`.

---

## 10. Backup y restore

### Backup (cron semanal sugerido)

```bash
#!/usr/bin/env bash
set -euo pipefail
STAMP=$(date +%F)
DEST=/backups/patriumhub
mkdir -p "$DEST"

mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" \
  --single-transaction --routines --triggers patriumhub \
  > "$DEST/patriumhub-$STAMP.sql"

cp /var/www/PatriumHub/PatriumHub/storage/app.key "$DEST/app.key-$STAMP"
chmod 600 "$DEST"/*
# retención ejemplo: borrar > 28 días
find "$DEST" -type f -mtime +28 -delete
```

- Dump y `app.key` en destinos distintos al DocumentRoot (ideal: otro disco / offsite).
- No subir backups a git.

### Restore

1. Mantener la app offline (Apache stop o `Require all denied` temporal).
2. `mysql -u root -p patriumhub < /backups/patriumhub/patriumhub-YYYY-MM-DD.sql`
3. Restaurar `app.key` con `chmod 640` y owner correcto.
4. Verificar `.env` y login.
5. Integraciones → Probar conexión.

Si se pierde `app.key`: hay que **re-cargar** tokens desde la UI; los cifrados previos no se descifran.

---

## 11. Checklist post-deploy

- [ ] HTTPS válido, HTTP redirige
- [ ] `APP_ENV=production`, `APP_DEBUG=false`, `SESSION_SECURE=true`
- [ ] DocumentRoot = `.../public` solamente
- [ ] Login OK; password seed cambiado
- [ ] `storage/app.key` generado y respaldado
- [ ] Cron sync + snapshots corren sin error
- [ ] Headers de seguridad presentes (DevTools → Network)
- [ ] Export CSV y toggle ocultar cifras OK
- [ ] Backup de prueba restaurable (al menos dry-run del dump)

---

## 12. Actualizar una versión ya desplegada

1. Poner mantenimiento breve si hay patches SQL.
2. `git pull` o rsync del código (sin pisar `.env` ni `app.key`).
3. Aplicar patches pendientes según release notes / carpeta `databases/`:

| Patch | Si schema es anterior a |
|-------|-------------------------|
| `patch_fase4.sql` | 0.4.0 |
| `patch_fase5.sql` | 0.5.0 |

```bash
mysql -u root -p patriumhub < databases/patch_fase5.sql
```

4. Verificar `settings.schema.version`.
5. Smoke: login, dashboard, sync de una integración.
6. Revisar logs Apache + `/var/log/patriumhub-*.log`.

---

## 13. Rollback rápido

1. Volver el código al tag/commit anterior (`git checkout` / rsync del artefacto previo).
2. Si el release tocó schema: restaurar dump SQL del backup previo + `app.key` de esa fecha.
3. Reload Apache.
4. Confirmar login y una sync.

No mezclar `app.key` de un backup con un dump de otra fecha si hubo rotación de clave.

---

## 14. Problemas frecuentes

| Síntoma | Qué revisar |
|---------|-------------|
| 404 en rutas amigables | `mod_rewrite`, `AllowOverride All`, `.htaccess` en `public/` |
| Página en blanco | `APP_DEBUG` temporal solo en staging; log de Apache; permisos |
| “No se pudo conectar a la base” | `DB_*` en `.env`, usuario MySQL, socket/host |
| Sync CLI falla por clave | Generar clave en Configuración; permisos de `storage/app.key` |
| Cookies de sesión no persisten | `APP_URL` HTTPS, `SESSION_SECURE`, reloj del server |
| Tokens ilegibles tras restore | Se restauró dump sin el `app.key` correcto |

---

## 15. Mapa de secretos (recordatorio)

| Secreto | Dónde | En git |
|---------|-------|--------|
| DB user/pass | `.env` | No |
| Clave cifrado | `storage/app.key` | No |
| Tokens MP / keys WC | BD cifrada (UI Integraciones) | No |
| Schema / patches | `databases/*.sql` | Sí |

---

## Ver también

- [09 — Guía de implementación](guia-implementacion.md) (local)
- [07 — Seguridad](07-seguridad-auth.md)
- [05 — Integraciones](05-integraciones.md)
- [10 — Plan de trabajo](plan-trabajo.md)
- [`../databases/apply-phpmyadmin.md`](../databases/apply-phpmyadmin.md)
