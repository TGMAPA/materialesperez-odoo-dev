# Materiales Pérez — Odoo Development

Entorno local de desarrollo para Odoo 19 Community.

## Componentes

- Odoo 19
- PostgreSQL 15
- `pos_multi_uom_price`: precios por unidad de medida en el POS.
- `pos_ticket_materialesperez`: personalización del ticket de venta.

## Requisitos

- Docker Engine
- Docker Compose
- Un archivo `.env` local con `ODOO_DB_PASSWORD`

## Iniciar el entorno

```bash
docker compose -p materialesperez-odoo-dev -f compose.dev.yml up -d

Abrir Odoo en:
http://127.0.0.1:18070

## Detener el entorno
```bash
docker compose -p materialesperez-odoo-dev -f compose.dev.yml down

Este comando no elimina los volúmenes nombrados, por lo que conserva la base de datos de desarrollo.

##Datos locales
La base de datos, el filestore, las credenciales y los respaldos no se versionan en Git.
Los datos necesarios para desarrollo deben restaurarse por separado desde un respaldo autorizado.

##Precauciones
- No conectar este entorno a la base de datos de producción.
- No publicar .env, dumps de PostgreSQL ni filestore.
- Probar cambios en desarrollo antes de desplegarlos en producción.
