# Migracion de Azure SQL Managed Instance a Azure SQL Database con DMS

## Escenario

Este procedimiento aplica al escenario seleccionado en Azure Database Migration Service (DMS):

```text
Origen: Azure SQL Managed Instance
Destino: Azure SQL Database
Modo: Offline
```

La migracion mediante DMS hacia Azure SQL Database es **offline**. No replica cambios de forma continua: la aplicacion debe detener escrituras antes de iniciar la migracion productiva y hasta que se complete el cambio al destino.

## Arquitectura

```text
Azure SQL Managed Instance ----> Self-hosted Integration Runtime ----> Azure SQL Database
```

El Self-hosted Integration Runtime (SHIR) ejecuta la copia de datos. Debe instalarse en una maquina Windows que pueda comunicarse con la Managed Instance origen y el destino Azure SQL Database.

> No usar el SHIR de Data Factory que ya esta registrado con otra clave. DMS crea o usa su propio SHIR y proporciona una clave de autenticacion diferente.

## 1. Crear una VM Windows temporal para el SHIR

Antes de crear recursos, validar que el asistente de DMS permita exactamente este par:

```text
Source server type: Azure SQL Managed Instance
Target server type: Azure SQL Database
Migration mode: Offline
```

Si ese par no aparece, detener esta configuracion: no se debe crear el SHIR hasta validar un metodo de migracion compatible.

### Red de la VM

1. En Azure Portal, abrir la VNet donde esta la Managed Instance.
2. Crear una subnet nueva para la VM, por ejemplo `snet-dms-ir`.
3. No usar la subnet delegada de la Managed Instance; esa subnet es exclusiva para recursos de Managed Instance.
4. Si la VM se crea en otra VNet, configurar VNet peering, rutas y NSG para permitir conectividad hacia la Managed Instance.

### Crear la VM

1. En Azure Portal, seleccionar **Virtual machines > Create > Azure virtual machine**.
2. Elegir imagen `Windows Server 2022 Datacenter`.
3. Seleccionar un tamano con al menos 4 vCPU, 8 GB de RAM y 80 GB disponibles en disco.
4. En **Networking**, seleccionar la subnet `snet-dms-ir`.
5. Usar Azure Bastion o RDP restringido a una IP de administracion. No exponer RDP a Internet sin restriccion.
6. Crear la VM y conectarse como administrador.

### Reglas de conectividad

Permitir desde la VM:

- Conexion a la Managed Instance por TCP 1433, usando su FQDN privado.
- Conexion a Azure SQL Database por TCP 1433 o al Private Endpoint configurado.
- Salida HTTPS por TCP 443 para que el Integration Runtime se registre y se comunique con Azure.

### Verificaciones de conectividad

En el host elegido, ejecutar estas pruebas. Sustituir los valores entre `< >`:

```powershell
Test-NetConnection <managed-instance-fqdn> -Port 1433
Test-NetConnection <servidor>.database.windows.net -Port 1433
```

## 2. Crear o abrir DMS

1. En Azure Portal, buscar **Azure Database Migration Service**.
2. Crear un recurso DMS o abrir el ya creado.
3. Confirmar que la suscripcion tiene registrado el proveedor `Microsoft.DataMigration`.
4. Confirmar que el recurso DMS esta en una region compatible y que el usuario tiene permisos sobre el destino.

## 3. Configurar el SHIR de DMS

1. En el recurso DMS, abrir **Settings > Integration runtime**.
2. Seleccionar **Configure integration runtime**.
3. Seleccionar **Download and install integration runtime**.
4. Copiar una de las claves de autenticacion que muestra DMS.
5. Descargar el instalador en el host Windows seleccionado en el paso 1.
6. Ejecutar el instalador.
7. Cuando abra **Microsoft Integration Runtime Configuration Manager**, pegar la clave de DMS.
8. Esperar la validacion y seleccionar **Register**.
9. Volver al portal DMS y esperar a que el nodo aparezca como `Online` en **Settings > Integration runtime**.

La clave de DMS no se debe guardar en GitHub, Dockerfile, YAML ni variables del App Service existente.

## 4. Preparar permisos

### En SQL Server origen

La cuenta de origen requiere, como minimo:

- Acceso a la base origen.
- Rol `db_datareader` sobre la base a migrar.
- Permiso de servidor `VIEW ANY DEFINITION`.

Para migrar esquema desde DMS, se requieren permisos adicionales, normalmente `db_owner` sobre la base origen. Validar los permisos exactos con el administrador de SQL Server antes de la ejecucion.

### En Azure SQL Database destino

1. Crear la base de datos destino con capacidad suficiente para la migracion.
2. Configurar red, firewall o Private Endpoint para que el SHIR alcance `<servidor>.database.windows.net:1433`.
3. Usar una cuenta con permisos de escritura sobre la base destino.
4. Si se migrara esquema mediante DMS, asignar los permisos de nivel servidor y base que solicite el asistente de DMS.

## 5. Ejecutar una prueba de migracion

1. En DMS, seleccionar **New migration**.
2. Seleccionar:
   - Source server type: `SQL Server`.
   - Target server type: `Azure SQL Database`.
   - Migration mode: `Offline`.
3. Seleccionar el SHIR que esta `Online`.
4. Proporcionar las credenciales del SQL Server origen y validar la conexion.
5. Proporcionar las credenciales del Azure SQL Database destino y validar la conexion.
6. Seleccionar las bases, esquema y tablas que se migraran.
7. Usar **Migrate missing schema** solo despues de revisar compatibilidad de objetos de SQL Server con Azure SQL Database.
8. Ejecutar primero con una base de QA o una muestra representativa.
9. En **Monitor migrations**, confirmar que todas las tablas terminen en estado `Succeeded`.

## 6. Validar la migracion de prueba

Validar antes de programar produccion:

1. Conteo de filas por tabla critica entre origen y destino.
2. Llaves primarias, indices, vistas, procedimientos almacenados y usuarios esperados.
3. Procesos funcionales de precios, etiquetas y fechas UTC/local.
4. Duracion real de la migracion y capacidad del Azure SQL Database destino.
5. Conectividad de la aplicacion al nuevo destino.

Registrar la duracion total: esa es la base para definir la ventana de indisponibilidad productiva.

## 7. Ejecucion productiva y corte

1. Confirmar respaldo y plan de reversa aprobados.
2. Comunicar la ventana de mantenimiento.
3. Detener las escrituras de la aplicacion en el SQL Server origen.
4. Confirmar que no existan transacciones pendientes.
5. Ejecutar la migracion DMS productiva.
6. Monitorear hasta el estado `Succeeded`.
7. Conciliar datos, precios, etiquetas y procesos criticos.
8. Cambiar la cadena de conexion de la aplicacion para apuntar a Azure SQL Database.
9. Validar operacion de la aplicacion.
10. Mantener el SQL Server origen como solo lectura durante el periodo de estabilizacion acordado.

## Deltas antes del corte

DMS hacia Azure SQL Database no ofrece una migracion online con deltas. Si se necesita mantener el destino actualizado antes del corte, usar Azure Data Factory con el SHIR de ADF ya desplegado:

1. Cargar el historico en Azure SQL Database.
2. Configurar un pipeline incremental con watermark, Change Tracking o CDC.
3. Probar dos ejecuciones consecutivas con cambios controlados.
4. En el corte, detener escrituras y ejecutar el ultimo delta.
5. Conciliar y redirigir la aplicacion al destino.

El SHIR de ADF y el SHIR de DMS pueden coexistir, pero usan claves de registro distintas y cumplen funciones diferentes.