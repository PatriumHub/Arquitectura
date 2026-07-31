# 11 — Guía de deploy

Deploy simple: **Apache en Linux** + import del SQL en **phpMyAdmin**.

Para jugar en XAMPP en la PC: [09 — Guía de implementación](guia-implementacion.md).

---

## 1. Subir el código

Dejá el repo en el server, por ejemplo `/var/www/PatriumHub`.

DocumentRoot de Apache → solo esto:

```
/var/www/PatriumHub/PatriumHub/public
```

---

## 2. Base de datos (phpMyAdmin)

1. Creá la BD `patriumhub` (utf8mb4) si el SQL no la crea solo.
2. **Importar** → `databases/patriumhub.sql`.
3. (Opcional, solo pruebas) `databases/seeds/demo_minimo.sql`.
4. Creá un usuario MySQL y dale permisos sobre `patriumhub`.

---

## 3. Config de la app

```bash
cd /var/www/PatriumHub/PatriumHub
cp .env.example .env
```

Editá `.env`:

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://tu-dominio.com

DB_HOST=127.0.0.1
DB_NAME=patriumhub
DB_USER=patrium_app
DB_PASS=tu_password
```

Sin tokens de Mercado Pago ni WooCommerce acá.

---

## 4. Apache (mínimo)

Vhost apuntando a `.../PatriumHub/public`, con `AllowOverride All` (hace falta el `.htaccess`).

HTTPS si podés (certbot o el que uses). Si es solo HTTP en red interna, poné `APP_URL` con `http://...` y no fuerces `SESSION_SECURE`.

---

## 5. Primer uso

1. Abrí la URL → login `admin@patriumhub.local` / `admin123`
2. Cambiá esa clave.
3. **Configuración** → generar clave de cifrado.
4. **Integraciones** → WC / MP si hace falta.

---

## 6. Cron (opcional)

Solo si usás sync automático:

```cron
*/30 * * * * php /var/www/PatriumHub/PatriumHub/cron/sync.php
15 3 * * * php /var/www/PatriumHub/PatriumHub/cron/snapshots.php
```

---

## 7. Backup rápido

Desde phpMyAdmin: exportar la BD.  
Guardá aparte `PatriumHub/storage/app.key` si ya generaste la clave de cifrado.

---

## Checklist

- [ ] DocumentRoot = `public/`
- [ ] SQL importado
- [ ] `.env` con DB + `APP_URL`
- [ ] Login OK + clave de cifrado generada
