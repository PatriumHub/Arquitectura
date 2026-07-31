# 09 — Guía de implementación

Guía operativa para levantar PatriumHub en **local / XAMPP** con Apache + MySQL/MariaDB + phpMyAdmin.

- Deploy en Apache Linux + phpMyAdmin: **[11 — Guía de deploy](guia-deploy.md)**  
- Import corto de BD: [`../databases/apply-phpmyadmin.md`](../databases/apply-phpmyadmin.md)

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

> En servidor Linux: [guía de deploy](guia-deploy.md).

---

## 6. Cron (opcional)

```bash
php cron/sync.php
php cron/snapshots.php
```

`sync.php` necesita la clave de Configuración. Seed demo: `databases/seeds/demo_minimo.sql`.

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

## 9. Backup

Exportar la BD desde phpMyAdmin. Si ya generaste la clave de cifrado, guardá también `storage/app.key`.
