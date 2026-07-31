# 09 — Guía de implementación

Guía operativa para levantar PatriumHub en **Apache + MySQL/MariaDB + phpMyAdmin**.  
Complementa el [plan de trabajo](plan-trabajo.md). Los pasos de código/SQL se ejecutan **después** de validar el plan.

---

## 1. Requisitos

| Componente | Mínimo |
|------------|--------|
| PHP | 8.1+ (extensiones: `pdo_mysql`, `openssl`, `mbstring`, `json`, `curl`) |
| Apache | con `mod_rewrite` |
| MySQL / MariaDB | 8.0+ / 10.4+ |
| phpMyAdmin | para importar `databases/patriumhub.sql` |

---

## 2. Repos y rutas

```
PatriumHub/                 # monorepo local
├── PatriumHub/             # código (DocumentRoot → public/)
├── databases/              # patriumhub.sql
└── Arquitectura/           # docs
```

En el servidor:

```
/var/www/PatriumHub/        # clone o copy del código
```

---

## 3. Base de datos (phpMyAdmin)

1. Crear base `patriumhub` con collation `utf8mb4_unicode_ci`.
2. Crear usuario MySQL dedicado (ej. `patrium_app`) con grants solo sobre `patriumhub`.
3. Importar `databases/patriumhub.sql`.
4. (Opcional) Importar `databases/seeds/demo_minimo.sql`.

No pegar tokens de Mercado Pago ni keys de WooCommerce en el SQL.

---

## 4. Config local de la app

Archivo sugerido (fuera de git o con ejemplo versionado):

```
DB_HOST=127.0.0.1
DB_NAME=patriumhub
DB_USER=patrium_app
DB_PASS=********
APP_KEY=base64:********
APP_URL=https://patriumhub.tudominio
```

- `APP_KEY` se genera una vez y **no se rota a la ligera** (cifra credentials en BD).
- No existen variables `MP_*` ni `WC_*` globales.

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

En producción: HTTPS (Let’s Encrypt u otro).

---

## 6. Cron de sincronización

```cron
*/30 * * * * php /var/www/PatriumHub/cron/sync.php >> /var/log/patriumhub-sync.log 2>&1
```

El cron solo procesa integraciones `active` cuyas credenciales están en BD.

---

## 7. Smoke test (post Fase 0/1)

- [ ] Login funciona  
- [ ] Crear persona y empresa  
- [ ] Crear cuenta con saldo  
- [ ] Dashboard muestra patrimonio  
- [ ] (Fase 2+) Alta WooCommerce desde UI → Probar conexión → Sync  
- [ ] (Fase 3+) Alta Mercado Pago desde UI → Probar conexión → Sync saldo  

---

## 8. Operación de integraciones (día a día)

1. Ir a **Integraciones**.  
2. Elegir proveedor.  
3. Completar entidad + credenciales.  
4. **Probar conexión**.  
5. **Sincronizar ahora** o esperar cron.  
6. Verificar dashboard de la entidad / MP.  
7. Si falla auth: rotar token/keys en la misma pantalla (no en el servidor).  

---

## 9. Backup

- Dump periódico de `patriumhub` (mysqldump o export phpMyAdmin).  
- Guardar `APP_KEY` en lugar seguro aparte del dump (sin la key no se descifran tokens).  
- Backups fuera del DocumentRoot.
