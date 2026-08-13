# 11 — Guía de deploy

URL: `http://192.168.100.50/patrium`

No hay que tocar Apache. Las rutas usan `/patrium/index.php?r=/...`.

---

## 1. Subir la app

Tiene que existir:

```
/var/www/html/patrium/index.php
```

```bash
sudo rm -rf /var/www/html/patrium
sudo mkdir -p /var/www/html/patrium
sudo cp -r /ruta/PatriumHub/* /var/www/html/patrium/
```

Prueba: `http://192.168.100.50/patrium/ok.php` → **PatriumHub OK**

---

## 2. SQL

phpMyAdmin → importar **solo** `databases/patriumhub.sql` (schema 0.8.2 + seed admin).  
No hace falta aplicar patches sueltos.  
Si la BD ya existía en 0.7.x: `scripts/migrate_v08.php?key=patrium-migrate-v08`.

---

## 3. `.env`

```env
APP_URL=http://192.168.100.50/patrium
DB_HOST=127.0.0.1
DB_NAME=patriumhub
DB_USER=root
DB_PASS=tu_clave_mysql
```

---

## 4. Entrar

`http://192.168.100.50/patrium/`

o

`http://192.168.100.50/patrium/index.php`

| Campo | Valor |
|-------|-------|
| Email | `admin@patriumhub.local` |
| Password | `admin123` |

Cambialo después del primer ingreso.
