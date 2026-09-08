# Guia de configuracion: VM como Distributor remoto entre Azure SQL Managed Instance y Azure SQL Database

## Objetivo

Configurar replicacion transaccional unidireccional desde Azure SQL Managed Instance (MI) hacia Azure SQL Database, usando una maquina virtual de Azure con SQL Server como Distributor remoto.

```text
Azure SQL Managed Instance (Publisher)
              |
              | TCP 1433
              v
Azure VM + SQL Server (Distributor remoto)
              |
              | TCP 1433
              v
Azure SQL Database (Push Subscriber)
```

La VM no funciona como puente pasivo: aloja la base `distribution` y ejecuta los agentes de Log Reader, Snapshot y Distribution. Azure SQL Database solo admite suscripciones de insercion (*push*) y no puede actuar como Publisher ni Distributor.

## Datos que se deben definir antes de iniciar

| Dato | Valor a completar |
| --- | --- |
| MI Publisher (FQDN) | `<mi-publicadora>` |
| Base de datos publicadora | `<base-origen>` |
| VM Distributor (FQDN) | `<vm-distributor>` |
| Instancia de SQL Server en VM | `<instancia-sql>` |
| Azure SQL Database Subscriber (FQDN) | `<servidor-destino>.database.windows.net` |
| Base de datos suscriptora | `<base-destino>` |
| Publicacion | `<publicacion>` |
| Azure File Share o recurso compartido de snapshot | `<ruta-snapshot>` |
| Retencion de distribucion | `<horas>` |
| Latencia maxima permitida | `<minutos o segundos>` |
| RPO y ventana de corte | `<valor aprobado>` |

## Requisitos tecnicos

### VM y SQL Server

1. Crear una VM Windows en Azure con SQL Server de version compatible e igual o posterior a la MI Publisher.
2. Habilitar e iniciar SQL Server Agent; los agentes de replicacion se ejecutan en la VM Distributor.
3. Dimensionar discos separados o suficientes para la base `distribution`, archivos de snapshot, registros y acumulacion durante fallas del Subscriber.
4. Definir respaldo, parcheo, monitoreo, acceso de emergencia y alta disponibilidad de la VM.
5. Usar una version actualizada de SSMS para administrar y monitorear la replicacion.

### Red

1. Configurar conectividad privada entre MI y VM: misma red virtual o peering/VPN, rutas, DNS y NSG validados.
2. Permitir VM Distributor hacia MI Publisher por TCP `1433`.
3. Permitir VM Distributor hacia Azure SQL Database por TCP `1433`.
4. Permitir VM Distributor hacia Azure Files por TCP `445` si se usa Azure File Share para snapshots.
5. Si Azure SQL Database usa Private Endpoint, asegurar que la VM resuelva el DNS privado correcto y tenga ruta hacia el endpoint.
6. Validar desde la VM las tres conexiones antes de crear la replicacion: MI, Azure SQL Database y almacenamiento de snapshots.

### Seguridad y esquema

1. Crear cuentas tecnicas de autenticacion SQL exclusivas para replicacion. No usar cuentas personales.
2. Guardar contrasenas, clave de Storage y la contrasena administrativa del Distributor en un secreto administrado, por ejemplo Azure Key Vault.
3. Crear previamente en Azure SQL Database el esquema compatible que la aplicacion requiere.
4. Verificar que cada tabla publicada tenga clave primaria.
5. Revisar objetos no compatibles en Azure SQL Database. No asumir la replicacion de permisos, particiones, indices filtrados, full-text, XML/XSD, espaciales o propiedades extendidas.

## Configuracion en QA

### 1. Preparar el Distributor remoto en la VM

1. Conectarse con SSMS a la instancia SQL Server de la VM.
2. En `Replication`, ejecutar `Configure Distribution`.
3. Seleccionar que esta instancia sera Distributor y crear la base `distribution`.
4. Definir el directorio raiz de snapshots. Se recomienda Azure File Share o un recurso compartido UNC administrado y accesible para los agentes de la VM.
5. Configurar la retencion de la base `distribution` de acuerdo con el tiempo maximo tolerado de indisponibilidad del destino.
6. Registrar la MI como Publisher remoto en el Distributor mediante `sp_adddistpublisher`, utilizando autenticacion SQL y una contrasena administrativa fuerte para el Distributor.

### 2. Asociar la MI con el Distributor remoto

1. Conectarse a la base `master` de la MI Publisher.
2. Ejecutar `sp_adddistributor` para registrar el FQDN de la VM Distributor y la misma contrasena administrativa configurada en el paso anterior.
3. Habilitar la base origen para publicacion transaccional mediante `sp_replicationdboption`.
4. Confirmar que la VM puede conectarse autenticadamente a la MI Publisher antes de continuar.

### 3. Crear la publicacion

1. En la base origen de MI, crear una publicacion transaccional.
2. Agregar exclusivamente las tablas aprobadas como articulos.
3. Revisar las opciones de esquema por articulo y deshabilitar las que no sean compatibles con Azure SQL Database.
4. Configurar Snapshot Agent y Log Reader Agent para ejecutarse en el Distributor remoto.
5. Documentar el nombre de la publicacion, agentes, frecuencia, retencion y credenciales asociadas.

### 4. Crear la suscripcion Push

1. Crear una suscripcion de tipo *push* desde la publicacion.
2. Indicar como Subscriber el FQDN de Azure SQL Database y el nombre de la base destino.
3. Usar autenticacion SQL para la conexion al Subscriber.
4. En T-SQL, usar `@subscriber_type = 0`, que es el unico valor admitido para Azure SQL Database.
5. Ejecutar Snapshot Agent para inicializar la suscripcion. Como alternativa, evaluar inicializacion por respaldo de MI cuando sea aplicable y este validada en QA.

## Pruebas de aceptacion

1. Confirmar que el snapshot crea o deja el destino consistente, segun las opciones de esquema elegidas.
2. Ejecutar inserciones, actualizaciones y eliminaciones controladas sobre cada tabla publicada.
3. Comparar conteos, claves, valores de negocio criticos e integridad referencial entre origen y destino.
4. Medir latencia extremo a extremo y compararla con el objetivo aprobado.
5. Detener y reiniciar Log Reader Agent y Distribution Agent; validar recuperacion sin huecos ni duplicados.
6. Simular indisponibilidad temporal de Azure SQL Database y medir crecimiento de `distribution` y del log de MI.
7. Probar el procedimiento de re-inicializacion de la suscripcion.
8. Ejecutar pruebas funcionales de la aplicacion contra Azure SQL Database.

## Corte y reversa

1. Mantener la replicacion activa hasta el momento de corte y monitorear errores, atraso, agentes, `distribution` y log de MI.
2. Detener las escrituras de negocio hacia MI.
3. Esperar a que Log Reader Agent y Distribution Agent procesen todos los comandos pendientes.
4. Conciliar origen y destino por ultima vez y confirmar atraso cero o dentro del limite aprobado.
5. Cambiar la cadena de conexion de la aplicacion a Azure SQL Database.
6. Mantener el destino sin escrituras de negocio durante el periodo de reversa acordado. Mientras no haya escrituras en el destino, la reversa consiste en volver la aplicacion a MI.

Despues de aceptar escrituras de negocio en Azure SQL Database no existe reversa automatica con esta topologia. Para reversa sin perdida, se requiere escritura dual idempotente o un mecanismo controlado para reprocesar cambios hacia MI.

## Operacion continua

1. Monitorear estado de Log Reader Agent, Snapshot Agent y Distribution Agent, latencia, errores y capacidad de la VM.
2. Alertar por agentes detenidos, atraso superior al objetivo, fallas de conexion y crecimiento anormal del log de MI o de `distribution`.
3. Mantener activo Log Reader Agent: mientras haya transacciones pendientes, el log de MI no podra truncarse.
4. Limpiar los archivos de snapshot cuando ya no sean necesarios.
5. Documentar y probar la reconstruccion de la replicacion ante falla de la VM, expiracion de retencion o failover de la MI.

## Fuentes tecnicas

- [Transactional replication - Azure SQL Managed Instance](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/replication-transactional-overview?view=azuresql)
- [Configure Publishing and Distribution](https://learn.microsoft.com/en-us/sql/relational-databases/replication/configure-publishing-and-distribution?view=sql-server-ver17)
- [Replication to Azure SQL Database](https://learn.microsoft.com/es-es/azure/azure-sql/database/replication-to-sql-database?view=azuresql)