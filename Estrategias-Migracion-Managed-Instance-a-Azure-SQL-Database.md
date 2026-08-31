# Estrategias de migracion: Azure SQL Managed Instance a Azure SQL Database

## Proposito

Este documento compara las estrategias para migrar una base desde Azure SQL Managed Instance hacia Azure SQL Database, considerando carga inicial, deltas, eliminaciones, reversa y la restriccion de no modificar la base origen.

```text
Origen: Azure SQL Managed Instance
Destino: Azure SQL Database
Restriccion principal: no alterar el origen sin aprobacion explicita
```

Antes de elegir una ruta, validar la compatibilidad de esquema y codigo con Azure SQL Database. Managed Instance ofrece capacidades de instancia que Azure SQL Database puede no admitir; ninguna herramienta corrige automaticamente esas diferencias.

## Criterios de decision

Evaluar cada alternativa con estas preguntas:

1. Es aceptable una ventana de indisponibilidad para la aplicacion?
2. El origen ya tiene `rowversion`, `FechaModificacion` u otra senal confiable de cambios?
3. Se necesita detectar eliminaciones fisicas antes del corte?
4. Se puede modificar la aplicacion para que escriba en dos destinos temporalmente?
5. Se puede habilitar una tecnologia de captura de cambios en el origen?
6. Se requiere reversa sin perdida despues de que el destino empiece a recibir escrituras?
7. El volumen permite leer tablas completas de forma recurrente?

## Comparativo

| Estrategia | Carga inicial | Deltas | Eliminaciones | Reversa despues de escrituras en destino | Cambios en origen | Uso recomendado |
|---|---|---|---|---|---|---|
| DMS offline | Si | No | Incluidas en la carga final | No automatica | No para deltas | Corte con ventana amplia |
| ADF con carga completa | Si | No reales | Mediante comparacion completa | Manual | No | QA o tablas pequenas |
| ADF con watermark existente | Si | Si | Solo si hay borrado logico existente | Manual | No | Origen con fecha o `rowversion` confiable |
| ADF con staging y comparacion | Si | Diferencias detectadas leyendo todo | Si | Manual | No | No hay watermark y el volumen lo permite |
| Escritura dual en aplicacion | Si | Si, en tiempo real | Si, si la aplicacion lo implementa | Buena | No en la base; si en la aplicacion | Reversa fuerte y cambio de aplicacion viable |
| Replicacion o CDC | Si | Si | Segun la tecnologia | Compleja | Normalmente si | Cuando se aprueban cambios en origen |
| Herramienta de terceros con CDC | Si | Si | Segun producto | Depende del producto | Normalmente si | Baja ventana y presupuesto/licencia disponibles |
| BACPAC o SqlPackage | Si | No | Incluidas en la exportacion | No automatica | No | Bases pequenas o migracion puntual |

## 1. Azure Database Migration Service (DMS) offline

DMS es una alternativa para una migracion puntual cuando el asistente soporte el par Azure SQL Managed Instance hacia Azure SQL Database. Para este destino, la estrategia debe asumirse como offline: no mantiene una sincronizacion continua de cambios hasta el corte.

Flujo:

1. Evaluar compatibilidad y corregir objetos no compatibles.
2. Ejecutar una migracion de QA y medir su duracion.
3. Programar una ventana de mantenimiento basada en esa medicion.
4. Detener escrituras en el origen.
5. Ejecutar la migracion, conciliar y cambiar la conexion de la aplicacion.

Ventajas: proceso guiado y adecuado para un corte unico.

Limites: no resuelve deltas previos al corte ni permite reversa sin perdida si ya se escribio en el destino.

## 2. Azure Data Factory (ADF)

ADF permite implementar una carga inicial y procesos recurrentes de copia mediante el Self-hosted Integration Runtime (SHIR). Es la alternativa base cuando se requiere conectar con la Managed Instance por red privada y se quiere evitar modificar el origen.

### ADF con watermark existente

Usar por tabla cuando ya exista una columna confiable como `FechaModificacion` o `rowversion`.

1. Cargar inicialmente la tabla completa al destino.
2. Guardar el watermark confirmado en una tabla de control del destino.
3. En cada corrida, capturar un limite superior en el origen y copiar el intervalo pendiente a staging.
4. Aplicar los cambios con un procedimiento idempotente.
5. Actualizar el watermark solo al completar copia, aplicacion y validacion.

Ventajas: reduce la lectura posterior y no modifica el origen.

Limites: una fecha o `rowversion` no suele detectar eliminaciones fisicas. No usar este mecanismo si la columna no se actualiza para todos los cambios relevantes.

### ADF con staging y comparacion completa

Usar cuando no exista una senal de cambios confiable y el volumen permita leer la tabla completa en cada corrida.

1. Copiar el origen completo a `stg.<Tabla>` en Azure SQL Database.
2. Comparar staging contra la tabla operativa por llave de negocio.
3. Registrar y aplicar `INSERT`, `UPDATE` y `DELETE` mediante un procedimiento almacenado.
4. Conservar auditoria de la ejecucion, conteos y diferencias detectadas.

Ventajas: detecta diferencias sin tocar la Managed Instance.

Limites: no es un delta real; su costo, duracion e impacto crecen con el volumen. Debe probarse en QA antes de usar en produccion.

La estrategia detallada de ADF esta en [ADF-Managed-Instance-a-Azure-SQL-Database.md](ADF-Managed-Instance-a-Azure-SQL-Database.md).

## 3. Escritura dual temporal desde la aplicacion

La aplicacion escribe cada alta, cambio y eliminacion tanto en Managed Instance como en Azure SQL Database durante una fase controlada de migracion.

Flujo:

1. Cargar el historico al destino mediante ADF, DMS u otra ruta validada.
2. Desplegar escritura dual con operaciones idempotentes y trazabilidad por solicitud.
3. Conciliar ambas bases durante el periodo de estabilizacion.
4. Cambiar las lecturas y escrituras para usar solo Azure SQL Database.
5. Conservar la capacidad de volver temporalmente al origen mientras ambas bases permanezcan sincronizadas.

Ventajas: permite deltas casi en tiempo real y ofrece la mejor reversa funcional sin tocar la configuracion de la base origen.

Riesgos: aumenta la complejidad de la aplicacion; se deben definir reintentos, orden de operaciones, transacciones fallidas, idempotencia y manejo de divergencias.

## 4. Change Data Capture (CDC) con ADF

CDC registra los cambios de las tablas habilitadas, incluidos `INSERT`, `UPDATE` y `DELETE`, y permite que ADF los lea como deltas. Es la alternativa recomendada para baja indisponibilidad cuando se autoriza modificar y operar la Managed Instance origen.

> Habilitar CDC modifica el origen: crea el esquema `cdc`, tablas de captura y trabajos de captura/limpieza. Requiere aprobacion formal del administrador de la Managed Instance, una prueba de capacidad y un plan de reversa.

### Ventajas

- Detecta inserciones, actualizaciones y eliminaciones fisicas; a diferencia de una fecha de modificacion, no depende de que la aplicacion actualice una columna.
- Reduce la lectura despues de la carga inicial porque ADF procesa los cambios capturados, no la tabla completa.
- Permite mantener el destino cercano al origen hasta el corte y ejecutar un delta final corto.
- Conserva el detalle y el orden de los cambios a traves de LSN, lo que facilita la auditoria y reprocesamiento controlado.

### Desventajas y riesgos

- Cambia la configuracion y consume recursos, almacenamiento y espacio de log en la Managed Instance origen.
- Requiere monitorear la latencia de captura y la retencion; si ADF se atrasa mas alla de la retencion, los cambios antiguos pueden dejar de estar disponibles y se debe recargar o reconciliar.
- Una tabla requiere llave primaria para habilitar `@supports_net_changes = 1`. Sin llave primaria, se debe redisenar la tabla o procesar cambios individuales segun la estrategia aprobada.
- CDC sincroniza en un sentido. No resuelve por si solo la reversa despues de aceptar escrituras en Azure SQL Database.
- Los cambios de esquema durante la sincronizacion deben gestionarse y probarse; no asumir que las nuevas columnas o cambios de tipo se propagan automaticamente.

### Prerrequisitos

1. Aprobar por escrito el cambio en el origen, la ventana de implementacion, el impacto de capacidad y la retencion de CDC.
2. Confirmar conectividad del SHIR de ADF con la Managed Instance y Azure SQL Database.
3. Identificar las tablas a capturar, sus llaves primarias y las columnas necesarias en el destino.
4. Implementar el esquema compatible, staging, tabla de control de LSN y procedimientos idempotentes en Azure SQL Database.
5. Ejecutar el procedimiento completo en QA con volumen y carga representativos.

### Paso a paso

#### 1. Habilitar CDC en la base origen

Ejecutar con una cuenta autorizada en la base de Azure SQL Managed Instance:

```sql
USE BaseOrigen;
GO

EXEC sys.sp_cdc_enable_db;
GO
```

Comprobar que CDC quedo habilitado:

```sql
SELECT name, is_cdc_enabled
FROM sys.databases
WHERE name = DB_NAME();
```

El resultado esperado es `is_cdc_enabled = 1`.

#### 2. Habilitar CDC por tabla

Habilitar solo las tablas incluidas en la migracion. El siguiente ejemplo requiere que `dbo.Clientes` tenga llave primaria:

```sql
EXEC sys.sp_cdc_enable_table
	@source_schema = N'dbo',
	@source_name = N'Clientes',
	@role_name = NULL,
	@supports_net_changes = 1;
GO
```

Repetir el procedimiento por cada tabla aprobada. Verificar las tablas y las instancias de captura:

```sql
SELECT
	s.name AS Esquema,
	t.name AS Tabla,
	t.is_tracked_by_cdc,
	c.capture_instance,
	c.supports_net_changes
FROM sys.tables AS t
INNER JOIN sys.schemas AS s ON s.schema_id = t.schema_id
LEFT JOIN cdc.change_tables AS c ON c.source_object_id = t.object_id
WHERE t.is_tracked_by_cdc = 1;
```

#### 3. Registrar el punto inicial y ejecutar la carga completa

No habilitar CDC despues de terminar la carga inicial: habria un intervalo sin capturar entre el fin de la copia y la activacion de CDC.

El orden correcto es:

1. Habilitar CDC en la base y en todas las tablas seleccionadas.
2. Registrar en la tabla de control del destino el LSN inicial de cada tabla.
3. Ejecutar la carga completa con ADF hacia staging y aplicar los datos al destino.
4. Conservar el LSN inicial; los cambios ocurridos mientras la carga completa estaba en proceso se aplican en el primer delta.

Ejemplo para obtener el LSN inicial de una instancia de captura:

```sql
SELECT sys.fn_cdc_get_min_lsn(N'dbo_Clientes') AS LsnInicial;
```

`dbo_Clientes` debe reemplazarse por el valor de `capture_instance` consultado en `cdc.change_tables`.

#### 4. Configurar los deltas en ADF

Para cada tabla, el pipeline debe:

1. Leer el LSN confirmado anterior desde la tabla de control del destino.
2. Obtener un LSN superior al inicio de la corrida para delimitar el intervalo.
3. Consultar los cambios CDC entre ambos LSN.
4. Copiar los resultados a staging en Azure SQL Database.
5. Ejecutar un procedimiento que aplique inserciones, actualizaciones y eliminaciones de forma idempotente.
6. Actualizar el LSN confirmado solamente si todas las actividades terminan correctamente.

ADF puede usar su funcionalidad CDC nativa o una Copy Activity con consulta parametrizada. Una consulta conceptual de cambios netos es:

```sql
SELECT *
FROM cdc.fn_cdc_get_net_changes_dbo_Clientes(
	@LsnAnterior,
	@LsnSuperior,
	N'all'
);
```

La funcion se genera a partir de la instancia de captura. El procedimiento del destino debe interpretar la columna `__$operation` para aplicar correctamente los cambios, en especial las eliminaciones.

#### 5. Monitorear y validar

1. Alertar ante fallas del pipeline, crecimiento anormal de log, latencia de CDC y proximidad al limite de retencion.
2. Registrar por corrida el LSN anterior, LSN superior, filas leidas, filas aplicadas, inicio, fin y resultado.
3. Comparar periodicamente conteos, llaves y valores criticos entre origen y destino.
4. Probar en QA inserciones, actualizaciones, eliminaciones, reintentos y cambios durante una carga completa.

#### 6. Ejecutar el corte y definir la reversa

1. Detener las escrituras de la aplicacion hacia la Managed Instance.
2. Esperar a que CDC capture los cambios pendientes y ejecutar el ultimo delta hasta el LSN final.
3. Conciliar las tablas y procesos criticos antes de redirigir la aplicacion.
4. Conservar la Managed Instance y sus respaldos durante el periodo de estabilizacion.
5. Si se requiere reversa despues de aceptar escrituras en Azure SQL Database, definir antes del corte escritura dual temporal, reproceso de operaciones o sincronizacion inversa previamente probada.

### Deshabilitar CDC

Solo deshabilitar CDC cuando ya no se necesite la sincronizacion y exista aprobacion. Primero se deshabilita por tabla y despues en la base:

```sql
EXEC sys.sp_cdc_disable_table
	@source_schema = N'dbo',
	@source_name = N'Clientes',
	@capture_instance = N'dbo_Clientes';
GO

EXEC sys.sp_cdc_disable_db;
GO
```

No deshabilitar CDC durante la fase de estabilizacion ni antes de confirmar que no se necesita una conciliacion adicional.

## 5. Replicacion transaccional

La replicacion transaccional es una alternativa de bajo desfase cuando la configuracion de publicacion, distribucion y suscripcion esta soportada y aprobada. Requiere validar de forma independiente el soporte exacto para Azure SQL Managed Instance como publicador y Azure SQL Database como suscriptor, los permisos, la red, la capacidad y el plan de reversa.

No es compatible con la regla actual de no modificar el origen salvo que exista una aprobacion formal. La sincronizacion suele ser en un solo sentido, por lo que tampoco resuelve automaticamente la reversa.

## 6. Herramientas de terceros basadas en captura de cambios

Productos de replicacion y CDC, por ejemplo Qlik Replicate, pueden reducir el trabajo de orquestacion y mantener el destino actualizado. Deben evaluarse como una solucion comercial, no como un reemplazo automatico de la arquitectura.

Validar antes de adquirir o implementar:

1. Soporte certificado para Azure SQL Managed Instance como origen y Azure SQL Database como destino.
2. Metodo de captura y cambios que exige en la Managed Instance.
3. Licenciamiento, infraestructura requerida, cifrado, credenciales y conectividad privada.
4. Latencia medida en QA, tratamiento de cambios de esquema, eliminaciones y conflictos.
5. Capacidad de detener, reiniciar y reconciliar sin perder datos.

Estas herramientas normalmente requieren CDC, lectura de logs o permisos/configuracion adicional en el origen; no asumir que cumplen la restriccion sin confirmarlo con el proveedor.

## 7. Exportacion e importacion con BACPAC o SqlPackage

Esta ruta exporta el esquema y los datos, e importa el resultado en Azure SQL Database. Puede servir para una base pequena, una prueba de compatibilidad o una migracion puntual con indisponibilidad.

Ventajas: no necesita un proceso de deltas y puede ser simple para un volumen reducido.

Limites: no mantiene el destino actualizado, la ventana de corte incluye el tiempo de exportacion/importacion y puede exponer incompatibilidades de esquema. No es adecuada cuando se requieren deltas o un corte de baja indisponibilidad.

## Estrategia de reversa

La reversa debe diseñarse antes del corte, no despues de un incidente.

1. Conservar la Managed Instance y sus respaldos; no eliminar ni degradar el origen al terminar la carga inicial.
2. Definir una ventana de estabilizacion y criterios de aceptacion funcionales y de datos.
3. Si falla antes de permitir escrituras en Azure SQL Database, detener el corte y restaurar la cadena de conexion al origen.
4. Si ya hubo escrituras en el destino, no volver al origen sin decidir como conservar esas operaciones: mantener el destino temporalmente en solo lectura, reprocesar operaciones en el origen o usar escritura dual/sincronizacion inversa previamente probada.
5. Registrar la hora del incidente, las transacciones afectadas, la decision de reversa y la conciliacion posterior.

La reversa mas segura sin cambios en el origen consiste en no aceptar escrituras en el destino hasta concluir las validaciones. La reversa mas fuerte despues de iniciar operaciones requiere escritura dual temporal o un mecanismo bidireccional probado, ambos con mayor complejidad.

## Recomendacion para este escenario

Con la restriccion actual de no modificar la Managed Instance:

1. Usar DMS offline solo si el negocio acepta detener escrituras durante toda la ventana de migracion.
2. Usar ADF con watermark para las tablas que ya tengan una senal de cambios confiable.
3. Usar ADF con staging y comparacion para tablas sin watermark, solo despues de medir su impacto en QA.
4. Para una reversa sin perdida despues de iniciar operaciones en el destino, evaluar escritura dual temporal desde la aplicacion.
5. Considerar replicacion, CDC o productos de terceros solamente si se aprueba modificar o configurar el origen y se completa una prueba de compatibilidad y rendimiento.