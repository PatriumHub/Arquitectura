# 09 — Guía de implementación

Guía operativa para levantar PatriumHub en **local / XAMPP** con Apache + MySQL/MariaDB + phpMyAdmin.

- Deploy en Apache Linux + phpMyAdmin: **[11 — Guía de deploy](guia-deploy.md)**  
- Import de BD: [`../databases/patriumhub.sql`](../databases/patriumhub.sql)

---

## 1. Requisitos

| Componente | Mínimo |
|------------|--------|
| PHP | 8.1+ (`pdo_mysql`, `openssl`, `mbstring`, `json`, `curl`) |
| Apache | PHP + MySQL (no hace falta `mod_rewrite`) |
| MySQL / MariaDB | 8.0+ / 10.4+ |
| phpMyAdmin | para importar `databases/patriumhub.sql` |

---

## 2. Repos y rutas

```
PatriumHub/                 # monorepo local
├── PatriumHub/             # código (copiar como /patrium en el server)
├── databases/              # patriumhub.sql (instalación única)
├── Guia_De_Uso/            # manual de usuario
└── Arquitectura/           # docs
```

En el servidor la app vive en `/var/www/html/patrium` → URL `http://IP/patrium`.  
Ver [guía de deploy](guia-deploy.md).

---

## 3. Base de datos (phpMyAdmin)

1. Importar **solo** `databases/patriumhub.sql` (crea BD + tablas + seed mínimo; schema 0.8.6).  
   No hace falta aplicar patches sueltos.
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

Usuario seed de la app (cambiar después del primer login):

| Campo | Valor |
|-------|-------|
| Email | `admin@patriumhub.local` |
| Password | `admin123` |

URL típica: `http://IP/patrium/` — las pantallas usan `index.php?r=/ruta` (sin rewrite).

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
APP_URL=http://192.168.100.50/patrium
APP_DEBUG=true
```

- En `.env` **solo** MySQL + URL. Sin tokens MP/WC.
- Entrás a `http://IP/patrium` → **Configuración** → Generar clave de cifrado.
- Después → **Integraciones** → + WooCommerce / + Mercado Pago.

---

## 5. Apache

Deploy en Linux: **[guía de deploy](guia-deploy.md)** (carpeta `patrium` con `index.php` adentro).

---

## 6. Cron (opcional)

```bash
php cron/sync.php
php cron/snapshots.php
```

`sync.php` necesita la clave de Configuración.

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
