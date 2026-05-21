# Manual de Explotación — WillmanTech S.L.

**Versión:** 1.0 | **Fecha:** 2026-05-19 | **Norma:** ISO/IEC/IEEE 26514:2022

---

## 1. Introducción y Arquitectura

WillmanTech S.L. gestiona sus ventas, facturas y clientes mediante un ERP desplegado en dos contenedores Docker:

- **web**: el ERP (puerto 8200, acceso vía navegador)
- **db**: la base de datos PostgreSQL (solo acceso interno)

Módulos activos: `account`, `sale`, `crm`, `purchase`, `stock` y `report`.

```
[ Navegador ] → puerto 8200 → [ Contenedor ERP ] → red interna → [ Contenedor PostgreSQL ]
```

---

## 2. Instalación

### Requisitos
- Docker y Docker Compose instalados
- 4 GB de RAM y 20 GB de disco libre como mínimo

### Pasos

**1.** Crear el fichero `.env`:

```env
POSTGRES_DB=willmantech_prod
POSTGRES_USER=erp_admin
POSTGRES_PASSWORD=CambiarEnProduccion123!
ERP_DB_HOST=db
ERP_DB_PORT=5432
HOST_PORT=8069
```

**2.** Crear el fichero `docker-compose.yml`:

```yaml
services:
  odoo:
    image: odoo:latest
    container_name: odoo
    restart: unless-stopped
    depends_on:
      - db
    ports:
      - "8200:8069"
    volumes:
      - odoo-data:/var/lib/odoo
      - ./config:/etc/odoo
      - ./addons:/mnt/extra-addons
    environment:
      - HOST=db
      - USER=odoo
      - PASSWORD=odoo
    command: odoo -d odoo --db_user=odoo --db_password=odoo -i base
  db:
      image: postgres:16.0
      container_name: db
      restart: unless-stopped
      environment:
        - POSTGRES_USER=odoo
        - POSTGRES_PASSWORD=odoo
        - POSTGRES_DB=odoo
        - PGDATA=/var/lib/postgresql/pgdata
      volumes:
        - db-data:/var/lib/postgresql/data

volumes:
  odoo-data:
  db-data:
```

**3.** Levantar el sistema:

```bash
docker compose up -d
```

**4.** Acceder a `http://localhost:8069` y crear la base de datos desde el asistente.

> **Importante:** El fichero `.env` no debe subirse a Git. Añadirlo al `.gitignore`.

---

## 3. Seguridad y Control de Acceso

El sistema define tres roles de usuario:

| Rol | Permisos |
|---|---|
| **Administrador** | Acceso total: configuración, usuarios y datos |
| **Contable** | Crear y validar facturas, consultar informes. Sin acceso a configuración |
| **Comercial** | Gestión de clientes y pedidos. Sin acceso a contabilidad |

### Política de contraseñas
- Mínimo 10 caracteres (mayúsculas, minúsculas, números y símbolos)
- Caducidad cada 90 días
- Bloqueo tras 5 intentos fallidos

Para crear un usuario: **Ajustes → Usuarios → Nuevo**, completar los datos y asignar el rol.

---

## 4. Backup y Restauración

### Crear copia de seguridad

```bash
docker exec willmantech_db pg_dump \
  -U erp_admin \
  -F c \
  -f /var/lib/postgresql/data/backup_$(date +%Y%m%d).dump \
  willmantech_prod
```

Copiar el fichero al host:

```bash
docker cp willmantech_db:/var/lib/postgresql/data/backup_20260506.dump /backups/
```

### Restaurar copia de seguridad

```bash
# 1. Parar el ERP
docker compose stop web

# 2. Restaurar el dump
docker exec -i willmantech_db pg_restore \
  -U erp_admin -d willmantech_prod \
  /var/lib/postgresql/data/backup_YYYYMMDD.dump

# 3. Arrancar el ERP
docker compose start web
```

---

## 5. Facturación y Generación de PDF

### Crear una factura

1. Ir a **Facturación → Clientes → Facturas → Nuevo**
2. Seleccionar el cliente y añadir las líneas (producto, cantidad, precio)
3. Pulsar **Confirmar** — se asigna un número definitivo (ej: `FACT/2026/00001`)
4. Pulsar **Imprimir** para obtener el PDF

### Pipeline de generación del PDF

Al pulsar "Imprimir", el sistema ejecuta el siguiente proceso:

```
Plantilla QWeb (XML)
        ↓
El motor QWeb inyecta los datos reales de la factura
        ↓
Se produce un documento HTML con el diseño final
        ↓
wkhtmltopdf convierte el HTML a PDF
        ↓
El PDF queda disponible para descarga o envío por correo
```

`wkhtmltopdf` actúa como un navegador headless que renderiza el HTML respetando los estilos CSS y lo exporta como PDF.

Si el PDF no se genera correctamente, revisar los logs del contenedor:

```bash
docker logs willmantech_erp --tail 50
```