# Estándar de despliegue — Materiales Pérez Odoo

## 1. Objetivo

Definir un procedimiento repetible y seguro para promover cambios de Odoo desde el entorno local de desarrollo hasta producción.

**Regla principal:** ningún cambio se prueba por primera vez en producción. Todo cambio debe probarse en desarrollo, revisarse, respaldarse y desplegarse de forma controlada.

## 2. Entornos

| Entorno | Uso | Acceso |
|---|---|---|
| Desarrollo | Modificar y validar código | `http://127.0.0.1:18070` |
| Producción | Operación real de Materiales Pérez | `http://192.168.10.101:8069` o la URL habitual del negocio |
| GitHub | Historial y distribución del código | Repositorio privado |

### Separación obligatoria

- Desarrollo utiliza su propia base PostgreSQL y sus propios datos/filestore.
- Producción utiliza exclusivamente sus servicios y datos productivos.
- Nunca conectar el contenedor de desarrollo a la base de datos de producción.
- Nunca ejecutar comandos de actualización o restauración sin confirmar primero el entorno, proyecto Compose y base de datos.
- No subir a Git contraseñas, archivos `.env`, dumps, filestore, sesiones ni datos sensibles.
- GitHub es control de versiones del código; **no sustituye los respaldos de la base de datos ni del filestore**.

## 3. Flujo general

1. Crear una rama de trabajo.
2. Implementar el cambio en desarrollo.
3. Probar funcionalidad y regresiones.
4. Revisar los cambios y comprobar que no haya secretos.
5. Hacer commit y subir la rama a GitHub.
6. Integrar los cambios aprobados en `main`.
7. Preparar y ejecutar un respaldo reciente de producción.
8. Desplegar el código aprobado en producción.
9. Actualizar únicamente los módulos afectados cuando sea necesario.
10. Ejecutar pruebas de humo y verificar los registros.
11. Documentar el resultado; si falla, detenerse y aplicar el plan de recuperación.

---

## 4. Desarrollo y pruebas

Trabajar desde el directorio local del proyecto:

```bash
cd ~/materialesperez-odoo-dev
```

### 4.1 Crear una rama para cada cambio

```bash
git switch main
git pull --ff-only
git switch -c feature/descripcion-breve
```

Usar nombres claros, por ejemplo:

- `feature/pos-ticket-taxes`
- `fix/pos-uom-price`
- `docs/deployment-process`

No realizar desarrollos directamente sobre `main`.

### 4.2 Iniciar y validar el entorno

```bash
docker compose -p materialesperez-odoo-dev -f compose.dev.yml up -d
docker compose -p materialesperez-odoo-dev -f compose.dev.yml ps
```

Abrir `http://127.0.0.1:18070` y probar el flujo afectado. Según el cambio, verificar:

- Inicio de sesión y carga de la base de desarrollo.
- Instalación/actualización del módulo.
- Flujo del Punto de Venta.
- Cálculos de precios, impuestos y totales, cuando aplique.
- Impresión del ticket.
- Productos, unidades de medida y demás funciones relacionadas.
- Registros de Odoo, sin errores relevantes.

Revisar logs:

```bash
docker compose -p materialesperez-odoo-dev -f compose.dev.yml logs --tail=200 web
```

Si se modifica un módulo Python o XML, comprobar también que Odoo cargue el módulo correctamente. No considerar validado un cambio solo porque la página abre.

### 4.3 Actualizar un módulo en desarrollo

Sustituir `<modulo>` por el nombre técnico del módulo que se está modificando:

```bash
docker compose -p materialesperez-odoo-dev -f compose.dev.yml exec web \
  odoo -d materialesperez -u <modulo> --stop-after-init \
  --db_host=db --db_user=odoo --db_password="$ODOO_DB_PASSWORD"
```

**Nota:** este ejemplo depende de que `ODOO_DB_PASSWORD` esté exportada en la terminal. No escribir la contraseña directamente en el comando. Si no está exportada, seguir el procedimiento local documentado para pasar la variable de forma segura. Después, revisar los logs y reiniciar el servicio web si corresponde.

Antes de ejecutar actualizaciones, confirmar el nombre de la base de datos y que el proyecto sea `materialesperez-odoo-dev`.

---

## 5. Revisión y control de versiones

Antes de guardar el cambio:

```bash
git status
git diff --check
git diff
```

Revisar que no se incluyan archivos de datos, credenciales, logs ni cambios ajenos a la tarea.

Comprobar explícitamente los archivos sensibles:

```bash
git check-ignore -v .env dev-materialesperez.dump dev-odoo-data
```

Si alguno de esos archivos ya estuviera versionado, `.gitignore` no lo elimina del historial automáticamente. Detenerse y corregirlo antes de publicar; si se expuso un secreto, rotarlo.

Cuando el cambio esté probado:

```bash
git add dev-custom-addons/ README.md
git diff --cached --check
git diff --cached
git commit -m "Descripción concreta del cambio"
git push -u origin feature/descripcion-breve
```

Agregar solo los archivos pertinentes. No usar `git add .` sin revisar previamente `git status`.

### Integración

- Abrir un Pull Request hacia `main`.
- Revisar el diff final y confirmar que las pruebas pasaron.
- Integrar el cambio aprobado.
- Mantener `main` como la versión aprobada para despliegue.
- Etiquetar cada versión desplegada con una etiqueta Git, por ejemplo `prod-2026-10-10-01`, y asociarla al commit exacto que se desplegó.

---

## 6. Lista de verificación previa a producción

No desplegar hasta confirmar todos los puntos aplicables:

- [ ] El cambio se probó en desarrollo.
- [ ] El commit que se desplegará está identificado.
- [ ] Se revisó el diff y no contiene secretos ni datos.
- [ ] Se identificaron los módulos que requieren actualización.
- [ ] Se verificó que no haya cambios de esquema o datos que requieran una migración especial.
- [ ] Se avisó a los usuarios y se eligió una ventana de bajo uso.
- [ ] Se dispone de un respaldo reciente de la base de datos **y** del filestore.
- [ ] Se comprobó que el respaldo terminó correctamente y que puede recuperarse.
- [ ] Se dispone de un plan de reversión y tiempo para ejecutarlo.
- [ ] Se confirmó la identidad del entorno productivo, su proyecto Compose y su base de datos.

**Si falta el respaldo, no se despliega.**

## 7. Respaldo de producción

Producción tiene un procedimiento de respaldo automatizado. Antes de cada despliegue:

1. Ejecutar o verificar una copia reciente mediante el procedimiento oficial del servidor productivo.
2. Confirmar que el proceso terminó sin errores.
3. Confirmar que el respaldo incluye la base de datos, el filestore, los addons personalizados y los archivos de configuración necesarios para recuperar el servicio.
4. Verificar la integridad del archivo y su ubicación.
5. Confirmar que se conoce el procedimiento de descifrado/restauración y que las claves necesarias están disponibles de forma segura.

No copiar contraseñas, claves privadas ni archivos de configuración secretos a este repositorio.

**Importante:** los comandos de respaldo y restauración deben ejecutarse en el servidor de producción siguiendo su script y documentación reales. No asumir que los nombres de servicios o rutas del entorno local son iguales a los de producción.

## 8. Despliegue a producción

Este procedimiento es deliberadamente conservador. Los nombres de servicio, rutas y comandos de producción deben confirmarse en el servidor antes de ejecutarlo.

### 8.1 Confirmar el commit aprobado

En el equipo donde se prepara el despliegue:

```bash
git switch main
git pull --ff-only
git log -1 --oneline
git status
```

Verificar que el commit sea exactamente el aprobado y que el árbol de trabajo esté limpio. Registrar el hash del commit.

### 8.2 Preparar el código en producción

Usar el mecanismo de despliegue aprobado para llevar **el commit exacto** a la ruta de addons personalizados de producción. No reemplazar toda la configuración de producción por `compose.dev.yml` ni copiar el filestore o el dump de desarrollo.

Antes de copiar archivos, confirmar:

- Ruta real de los addons personalizados.
- Usuario y permisos correctos.
- Dependencias requeridas.
- Compatibilidad con la versión de Odoo instalada.
- Que no se sobrescriban archivos de configuración o secretos de producción.

Preferir una copia controlada de los módulos versionados frente a sincronizar todo el directorio del proyecto. Revisar el comando de copia antes de ejecutarlo, especialmente si incluye opciones para eliminar archivos de destino.

### 8.3 Ventana de mantenimiento

1. Avisar a los usuarios que el POS y Odoo pueden no estar disponibles temporalmente.
2. Confirmar el respaldo previo.
3. Seguir el procedimiento operativo específico del servidor para reducir o detener temporalmente el tráfico, si es necesario.
4. Desplegar el código aprobado.
5. Actualizar únicamente los módulos que requieran cambios de esquema o datos.
6. Revisar los logs.
7. Reiniciar los servicios solo si es necesario y mediante el procedimiento productivo documentado.

**No ejecutar a ciegas comandos de `docker compose down`, restauraciones o actualizaciones sobre producción.** El nombre del proyecto Compose y los servicios deben identificarse primero en el servidor productivo. No utilizar el proyecto `materialesperez-odoo-dev` para operar producción.

### 8.4 Actualización de módulos

Si el cambio requiere actualizar un módulo, ejecutar el comando de actualización en el entorno productivo usando su procedimiento aprobado, las credenciales suministradas de forma segura y la base de datos productiva confirmada.

No actualizar todos los módulos por defecto. Actualizar solo los módulos afectados y sus dependencias cuando corresponda.

Si la actualización falla:

- No repetirla indefinidamente.
- No borrar volúmenes ni datos.
- Guardar el error y los logs.
- Detener el despliegue y evaluar recuperación.

---

## 9. Pruebas posteriores al despliegue

Realizar y registrar estas pruebas según el alcance del cambio:

- [ ] La URL habitual de producción responde.
- [ ] Se puede iniciar sesión.
- [ ] El Punto de Venta carga correctamente.
- [ ] Se pueden consultar productos y precios.
- [ ] Los cálculos de precios, impuestos y totales son correctos.
- [ ] El ticket se imprime y conserva el formato esperado.
- [ ] Los flujos relacionados con el cambio funcionan.
- [ ] Los logs no muestran errores nuevos críticos.
- [ ] Los usuarios responsables confirman que la operación normal continúa.

Si falla una prueba crítica, considerar el despliegue no exitoso y activar el plan de recuperación.

## 10. Recuperación ante fallos

La recuperación debe seguir el procedimiento probado para el servidor productivo:

1. Detener nuevos cambios y registrar el commit desplegado, la hora y el error.
2. Determinar si el problema se puede corregir de forma segura con un cambio pequeño. No improvisar cambios de esquema.
3. Si es necesario revertir, volver a la versión de código anterior y, cuando corresponda, restaurar **en conjunto** la base de datos y el filestore desde un respaldo compatible.
4. Seguir el procedimiento de recuperación documentado, verificando el destino antes de restaurar para no sobrescribir datos equivocados.
5. Validar inicio de sesión, POS, productos, tickets y logs.
6. Documentar la causa y las acciones correctivas antes de volver a desplegar.

**Un `git revert` solo revierte cambios de código. No revierte automáticamente cambios en la base de datos, el filestore ni los datos ya modificados por un módulo.** Las migraciones y actualizaciones de módulos pueden no ser reversibles sin restaurar un respaldo.

---

## 11. Registro de despliegues

Mantener un registro por despliegue con estos datos:

| Campo | Qué registrar |
|---|---|
| Fecha y hora | Inicio y fin, con zona horaria |
| Responsable | Persona que ejecutó y validó |
| Commit | Hash y etiqueta Git |
| Módulos | Módulos desplegados/actualizados |
| Respaldo | Identificador o ubicación protegida del respaldo |
| Resultado | Exitoso, fallido o revertido |
| Validación | Pruebas realizadas y resultado |
| Incidencias | Errores, impacto y acciones correctivas |

No incluir contraseñas, tokens ni datos personales de clientes en el registro.

## 12. Reglas no negociables

1. Producción nunca se utiliza como entorno de prueba.
2. No desplegar sin respaldo verificable.
3. No versionar secretos, dumps ni filestore.
4. No desplegar cambios sin commit identificable.
5. No usar configuración de desarrollo en producción.
6. No actualizar módulos sin identificar la base de datos y el entorno.
7. No considerar exitoso un despliegue hasta completar las pruebas posteriores.
8. Todo incidente y toda recuperación deben documentarse.
