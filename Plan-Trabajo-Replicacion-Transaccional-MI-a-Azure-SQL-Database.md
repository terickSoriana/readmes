# Plan de trabajo: replicacion transaccional de Azure SQL Managed Instance a Azure SQL Database

## Objetivo

Migrar las tablas de negocio desde Azure SQL Managed Instance (MI) hacia Azure SQL Database con una carga inicial y replicacion transaccional continua. El objetivo es mantener el destino actualizado antes del corte, reducir la ventana de indisponibilidad y conservar una ruta de reversa mientras el destino no reciba escrituras de negocio.

```text
Azure SQL Managed Instance (Publisher)
              |
              v
Distributor
              |
              v
Azure SQL Database (Push Subscriber)
```

Azure SQL Database solo puede actuar como suscriptor de insercion (*push*) en esta topologia. La sincronizacion es unidireccional: no cubre la reversa despues de que se acepten escrituras en el destino.

## Decision de arquitectura

### Opcion recomendada: distribuidor local en la Managed Instance

Configurar la misma MI como **Publisher** y **Distributor**. Es la topologia mas simple para una unica MI origen y un unico destino Azure SQL Database: reduce componentes, eliminando la operacion de una VM y la comunicacion adicional entre publicador y distribuidor.

Los archivos de snapshot se almacenan en un recurso compartido de Azure Storage, no en un recurso local de la MI.

### Opcion alternativa: VM como distribuidor remoto

Si, se puede usar una VM de Azure con SQL Server como componente de comunicacion, pero tecnicamente debe operar como un **Distributor remoto**, no como un puente pasivo ni como una instancia de ADF.

```text
Azure SQL Managed Instance (Publisher)
              |
              v
Azure VM con SQL Server (Distributor remoto)
              |
              v
Azure SQL Database (Push Subscriber)
```

Esta opcion es valida si se necesita aislar la carga de agentes de replicacion, centralizar distribucion para varios publicadores o se cuenta con operacion DBA establecida para SQL Server en VM. Agrega costo, mantenimiento, respaldos, parcheo, alta disponibilidad y monitoreo de la VM. El SQL Server de la VM debe tener una version compatible e igual o posterior a la del publicador, y tanto MI como la VM deben estar en la nube. Si sus redes virtuales son distintas, se requiere conectividad privada entre ellas mediante peering o VPN.

No seleccionar la VM solo para "conectar" ambas bases: la MI puede comunicarse directamente con Azure SQL Database por TCP 1433. Elegir la VM unicamente si se justifica como distribuidor remoto.

## Alcance y criterios de aceptacion

El alcance inicial debe limitarse a las tablas aprobadas para la migracion. Antes del corte se debe demostrar en QA que:

1. La carga inicial crea un destino consistente.
2. Las inserciones, actualizaciones y eliminaciones llegan al destino en el orden esperado.
3. La latencia permanece dentro del objetivo acordado.
4. Una interrupcion del agente no genera perdida ni duplicacion de datos al recuperarse.
5. El corte finaliza dentro de la ventana aprobada.
6. La aplicacion funciona sobre Azure SQL Database y la reversa funciona mientras el destino siga sin escrituras de negocio.

## Fase 0: aprobacion y diseno

1. Confirmar que Azure SQL Database soporta el esquema, objetos, tipos de datos y comportamiento funcional requerido por la aplicacion.
2. Inventariar tablas, llaves primarias, volumen, tasa de cambios, dependencias, objetos no compatibles y prioridad de migracion.
3. Definir el RPO, la latencia maxima permitida y la ventana de corte.
4. Elegir y aprobar la topologia: MI como distribuidor local o VM con SQL Server como distribuidor remoto.
5. Definir responsables para la MI, Azure SQL Database, red, Azure Storage, aplicacion, validacion funcional y monitoreo.
6. Acordar retencion de distribucion, frecuencia de agentes, alertas y procedimiento de re-inicializacion.

## Fase 1: preparar destino y red

1. Crear Azure SQL Database con capacidad suficiente y aplicar el esquema compatible antes de inicializar la suscripcion.
2. Crear usuarios y credenciales de autenticacion SQL exclusivos para replicacion, con permisos minimos necesarios. No usar credenciales personales.
3. Crear una cuenta de Azure Storage y un Azure File Share para los snapshots cuando el distribuidor sea MI. Proteger la clave de almacenamiento en un secreto administrado.
4. Abrir la salida TCP 445 desde la subred de la MI hacia Azure Files.
5. Abrir la salida TCP 1433 desde la MI distribuidora o desde la VM distribuidora hacia Azure SQL Database. Ajustar las reglas NSG de MI necesarias para salida hacia el destino.
6. Si se usa VM remota, habilitar conectividad privada MI a VM por TCP 1433 y validar DNS, rutas, NSG y peering/VPN antes de configurar replicacion.
7. Verificar desde el futuro distribuidor la conexion autenticada a la MI publicadora, Azure Storage y Azure SQL Database.

## Fase 2: configurar publicacion y distribucion en QA

1. Usar la version mas reciente de SSMS para administrar la replicacion; preferir scripts T-SQL versionados para que la configuracion sea repetible.
2. Configurar el Distributor y crear la base `distribution`.
3. Registrar la MI como Publisher en el Distributor.
4. Para MI como distribuidor local, configurar `@working_directory` con Azure File Share y `@storage_connection_string` con la clave de Azure Storage. No usar rutas UNC locales de ejemplos de SQL Server.
5. Habilitar solo la base origen aprobada para publicacion transaccional mediante `sp_replicationdboption`.
6. Crear la publicacion transaccional e incluir unicamente los articulos aprobados. Cada tabla debe tener clave primaria.
7. Definir opciones de snapshot y de esquema de acuerdo con las limitaciones de Azure SQL Database; no asumir que permisos, particiones, indices filtrados, XML, espaciales, full-text u otras propiedades no compatibles se replicaran.
8. Crear una suscripcion *push* en Azure SQL Database usando autenticacion SQL.
9. Ejecutar el snapshot e inicializar la suscripcion. Desde MI tambien puede evaluarse inicializacion por respaldo cuando aplique.
10. Registrar scripts, nombres de publicaciones, agentes, credenciales, endpoints, retencion y responsables en el runbook operativo.

## Fase 3: pruebas de QA

1. Realizar una carga inicial con volumen representativo y registrar duracion, capacidad usada en MI, almacenamiento de snapshot y costo estimado.
2. Generar inserciones, actualizaciones y eliminaciones controladas en todas las tablas publicadas.
3. Verificar conteos, llaves, valores criticos, integridad referencial y la latencia extremo a extremo.
4. Detener y reanudar Log Reader Agent y Distribution Agent; confirmar recuperacion, alertas y ausencia de duplicados o huecos.
5. Simular indisponibilidad temporal de Azure SQL Database y medir la acumulacion en distribucion y en el log de MI.
6. Verificar que el Log Reader Agent este activo: mientras existan transacciones pendientes de replicar, el log de transacciones no puede truncarse.
7. Probar una modificacion de esquema aprobada o confirmar el congelamiento de esquema durante toda la sincronizacion.
8. Ejecutar pruebas funcionales de la aplicacion contra Azure SQL Database.
9. Documentar el resultado y obtener aprobacion de QA, DBA, red y negocio antes de produccion.

## Fase 4: preparacion productiva

1. Repetir la configuracion validada en QA mediante los scripts aprobados.
2. Ejecutar el snapshot o la inicializacion por respaldo y validar que todas las suscripciones esten activas.
3. Mantener la replicacion en ejecucion antes del corte y monitorear latencia, errores, estado de agentes, crecimiento del log de MI y retencion de `distribution`.
4. Conciliar periodicamente origen y destino por conteos, llaves y valores de negocio criticos.
5. Eliminar de Azure Storage los archivos de snapshot que ya no sean necesarios; MI no los borra automaticamente.
6. Congelar cambios de esquema y cambios no esenciales de aplicacion hasta completar el corte y estabilizacion.
7. Confirmar respaldos, acceso de emergencia, plan de comunicacion, responsables y criterios de abortar el corte.

## Fase 5: corte productivo

1. Confirmar que la replicacion no tiene errores y que el atraso esta dentro del limite acordado.
2. Detener las escrituras de negocio hacia la MI y esperar las transacciones pendientes.
3. Esperar a que Log Reader Agent y Distribution Agent apliquen el ultimo conjunto de cambios en Azure SQL Database.
4. Confirmar que no quedan comandos sin distribuir y ejecutar la conciliacion final.
5. Cambiar la cadena de conexion de la aplicacion a Azure SQL Database.
6. Ejecutar pruebas funcionales y monitorear errores, rendimiento y operaciones de negocio criticas.
7. Mantener la MI como referencia durante el periodo de estabilizacion definido.

## Reversa

Antes de aceptar escrituras de negocio en Azure SQL Database, la reversa es cambiar la aplicacion nuevamente hacia MI y detener o pausar la replicacion segun el procedimiento aprobado.

Despues de aceptar escrituras en Azure SQL Database no existe reversa automatica con esta topologia, porque Azure SQL Database no admite suscripcion actualizable ni replicacion bidireccional. Si se exige reversa sin perdida despues del corte, implementar y probar antes una de estas medidas:

1. Mantener el destino en solo lectura durante el periodo de validacion.
2. Implementar escritura dual temporal e idempotente en la aplicacion.
3. Registrar y reprocesar las operaciones nuevas hacia MI mediante un procedimiento controlado.

## Operacion posterior al corte

1. Mantener monitoreo de agentes, errores, latencia, capacidad de `distribution`, crecimiento de log y almacenamiento de snapshots mientras la replicacion siga activa.
2. Definir umbrales y alertas para agentes detenidos, atraso de distribucion, errores de conexion y crecimiento anormal del log.
3. Si se usan grupos de conmutacion por error de MI, documentar y probar la limpieza y reconfiguracion de publicaciones despues de un failover del Publisher o Distributor.
4. Tras cerrar la estabilizacion y aprobar la retirada del origen, detener agentes, eliminar suscripciones y publicaciones de forma controlada, y limpiar snapshots residuales de Azure Storage.

## Fuentes tecnicas

- [Transactional replication - Azure SQL Managed Instance](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/replication-transactional-overview?view=azuresql)
- [Configure Publishing and Distribution](https://learn.microsoft.com/en-us/sql/relational-databases/replication/configure-publishing-and-distribution?view=sql-server-ver17)
- [Replication to Azure SQL Database](https://learn.microsoft.com/es-es/azure/azure-sql/database/replication-to-sql-database?view=azuresql)