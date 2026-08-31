# ADF: Azure SQL Managed Instance a Azure SQL Database

## Proposito

Esta guia corresponde exclusivamente a Azure Data Factory (ADF) y al Self-hosted Integration Runtime (SHIR) desplegado en el contenedor.

```text
Origen: Azure SQL Managed Instance
Destino: Azure SQL Database
Objetivo: movimiento de datos y, solo si es posible, cargas incrementales
```

## Que hace ADF

ADF permite crear linked services al origen y destino, definir datasets y ejecutar actividades de copia. El SHIR ejecuta las conexiones hacia recursos que requieren red privada.

ADF no es una herramienta de migracion completa de instancia: no replica automaticamente configuraciones de servidor, logins, SQL Agent Jobs u otros objetos fuera de las tablas y el esquema que se implemente de forma explicita.

## Configuracion basica

1. En ADF Studio, crear un linked service para Azure SQL Managed Instance y seleccionar el SHIR operativo.
2. Crear un linked service para Azure SQL Database. Seleccionar el SHIR solo si el destino usa conectividad privada; en caso contrario, usar el Integration Runtime adecuado de Azure.
3. Ejecutar **Test connection** en ambos linked services.
4. Crear datasets para las tablas origen y destino.
5. Crear un pipeline con Copy Activity.
6. Ejecutar primero una carga completa en QA y validar conteos y llaves.

## Deltas y la restriccion de no modificar el origen

ADF solo puede hacer una carga delta si hay una forma confiable de identificar los cambios. Ejemplos: una columna existente de ultima modificacion, `rowversion`, Change Tracking o CDC.

En este proyecto, la base origen no puede modificarse. Por tanto:

- No habilitar Change Tracking o CDC.
- No agregar triggers, columnas de marca de agua ni tablas auxiliares en Managed Instance.
- No afirmar que existe delta solo porque el SHIR muestra `Running`.

Si las tablas ya tienen una columna confiable de actualizacion o un `rowversion` existente, ADF puede consultarla sin modificar la base. Esa posibilidad debe validarse por tabla con el responsable de datos.

Si no existe una señal de cambios ya disponible, ADF solo puede hacer copias completas. Una copia completa recurrente no es una replicacion delta y debe evaluarse por volumen, costo, duracion e impacto en el origen.

## Alternativa: detectar diferencias en el destino

Como la base origen no se puede modificar, es posible agregar tablas auxiliares en **Azure SQL Database destino** para comparar el estado actual contra una nueva copia del origen. Esta alternativa no modifica la Managed Instance, pero ADF debe leer la tabla completa del origen en cada ejecucion. Por ello, no es un delta real en el origen.

Flujo propuesto:

1. ADF copia la tabla completa de origen a una tabla de staging, por ejemplo `stg.PreciosOrigen`.
2. Un procedimiento almacenado en el destino compara staging contra la tabla operativa usando la llave de negocio, por ejemplo `ProductoId` y `TiendaId`.
3. El procedimiento registra las diferencias en una tabla de auditoria: valor anterior, valor nuevo, fecha de deteccion y tipo de cambio (`INSERT`, `UPDATE` o `DELETE`).
4. El procedimiento aplica en la tabla operativa solamente los registros que cambiaron.
5. Staging se limpia o se reemplaza antes de la siguiente ejecucion.

Ejemplo de tabla de auditoria en Azure SQL Database:

```sql
CREATE TABLE auditoria.CambiosPrecio (
	ProductoId int NOT NULL,
	TiendaId int NOT NULL,
	MontoAnterior decimal(18, 2) NULL,
	MontoNuevo decimal(18, 2) NULL,
	TipoCambio varchar(10) NOT NULL,
	FechaDeteccion datetime2 NOT NULL DEFAULT sysutcdatetime()
);
```

Ventajas:

- Respeta la regla de no cambiar la base origen.
- Conserva una bitacora de los cambios detectados.
- Permite aplicar al destino solo inserciones, actualizaciones y eliminaciones identificadas.

Riesgos y limites:

- Cada ejecucion lee todos los registros de la tabla origen.
- La carga, costo y duracion crecen con el volumen de datos.
- La comparacion requiere llaves de negocio estables y reglas claras para valores nulos, registros eliminados y conflictos.
- Se debe medir en QA antes de usarlo en produccion.

## Prueba de delta sin modificar el origen

Solo realizar esta prueba si existe una columna de cambio ya implementada:

1. Documentar la columna disponible y comprobar que se actualiza para todos los cambios relevantes.
2. Ejecutar una primera copia y guardar el ultimo valor visto en una tabla de control del **destino**, no del origen.
3. Realizar cambios controlados en QA mediante el proceso autorizado de la aplicacion.
4. Ejecutar una segunda copia filtrando por el intervalo de valores de la columna existente.
5. En **Monitor > Activity runs**, validar que `rowsRead` y `rowsCopied` coincidan con los cambios controlados.
6. Validar en Azure SQL Database que no existan duplicados y que las actualizaciones se apliquen correctamente.

Si la prueba no es posible o no detecta todas las modificaciones y eliminaciones, no usar ADF como mecanismo de delta para el corte.

## Estrategia operativa: carga inicial, deltas y corte

La estrategia debe definirse por tabla. Antes de construir los pipelines, documentar para cada tabla la llave de negocio, el volumen, la columna de cambios disponible, si permite identificar eliminaciones, la frecuencia requerida y el metodo de carga.

```text
Azure SQL Managed Instance
  -> ADF con SHIR
  -> stg.Tabla en Azure SQL Database
  -> Procedimiento almacenado idempotente
  -> Tabla operativa + control de watermarks + auditoria
```

### 1. Preparacion del destino

1. Implementar en Azure SQL Database el esquema compatible: tablas, llaves, indices, vistas y procedimientos adaptados.
2. Crear los esquemas `stg`, `ctl` y `auditoria` en el destino.
3. Crear una tabla de control en `ctl` por tabla replicada. Debe conservar como minimo el watermark confirmado, la fecha de ejecucion y el identificador de la corrida de ADF.
4. Crear una tabla de staging por tabla de origen y un procedimiento almacenado que aplique los datos hacia la tabla operativa.
5. Definir permisos minimos: lectura en el origen y escritura, ejecucion de procedimientos y administracion de staging en el destino.

### 2. Carga inicial completa

1. Ejecutar una Copy Activity por tabla desde la Managed Instance hacia `stg.<Tabla>`.
2. Al terminar cada copia, ejecutar un Stored Procedure en Azure SQL Database para insertar o actualizar desde staging hacia la tabla operativa.
3. Registrar por tabla las filas leidas, copiadas, insertadas, actualizadas, la duracion y el resultado de la corrida.
4. Conciliar conteos, llaves duplicadas, valores nulos criticos y reglas funcionales antes de declarar lista cada tabla.
5. Limpiar o reemplazar staging unicamente despues de que la conciliacion y el procedimiento terminen correctamente.

La carga inicial debe probarse en QA con volumen representativo. Su duracion define si conviene particionar tablas grandes y la ventana necesaria para el corte final.

### 3. Delta con una señal existente de cambios

Usar este flujo solo cuando la tabla tenga una columna existente y confiable, por ejemplo `FechaModificacion` o `rowversion`.

1. Consultar en el origen un limite superior al inicio de la corrida, por ejemplo `MAX(FechaModificacion)` o el mayor `rowversion` disponible.
2. Leer solamente el intervalo entre el watermark confirmado y ese limite superior.
3. Copiar los cambios a staging y ejecutar el procedimiento idempotente en el destino.
4. Actualizar el watermark en `ctl` solamente cuando la copia, la aplicacion y las validaciones terminen correctamente.
5. Ante una falla, conservar el watermark anterior para reprocesar el intervalo completo sin perder cambios.

Ejemplo conceptual para una columna de fecha:

```sql
WHERE FechaModificacion > @WatermarkAnterior
	AND FechaModificacion <= @WatermarkSuperior
```

El uso de un limite superior fijo evita que cambios que lleguen durante la ejecucion queden mezclados en la misma corrida. Esos cambios se procesan en la siguiente ejecucion.

### 4. Eliminaciones y tablas sin watermark

Las columnas `FechaModificacion` y `rowversion` normalmente no detectan eliminaciones fisicas. Si existe una marca de borrado logico ya implementada, incluirla en el delta. Si no existe, las eliminaciones solo se detectan mediante una comparacion completa o durante el corte con las escrituras detenidas.

Para tablas sin una senal de cambios confiable:

1. Copiar la tabla completa a staging en cada corrida.
2. Comparar staging con la tabla operativa mediante la llave de negocio.
3. Registrar en auditoria los `INSERT`, `UPDATE` y `DELETE` detectados.
4. Aplicar solo esas diferencias mediante un procedimiento idempotente.

Este mecanismo no modifica el origen, pero no es un delta real porque vuelve a leer toda la tabla en cada ejecucion. Debe aprobarse segun volumen, costo, duracion e impacto sobre la Managed Instance.

### 5. Frecuencia, monitoreo y reintentos

1. Programar la frecuencia de cada pipeline segun el volumen, la duracion real y el desfase aceptado por el negocio, por ejemplo cada 15, 30 o 60 minutos.
2. Configurar alertas para fallas, duracion anormal, retraso del watermark y diferencias de conteo.
3. Conservar el identificador de corrida de ADF, el watermark anterior y superior, las filas leidas y las filas aplicadas para poder auditar cada ejecucion.
4. Disenar el procedimiento de aplicacion para que un reintento del mismo intervalo no genere duplicados ni actualizaciones inconsistentes.

### 6. Corte productivo

1. Confirmar respaldo, plan de reversa y ventana de mantenimiento aprobados.
2. Detener las escrituras de la aplicacion hacia la Managed Instance y esperar transacciones pendientes.
3. Ejecutar el ultimo delta; si una tabla no tiene delta confiable, ejecutar su carga completa final y comparacion en staging.
4. Conciliar las tablas y procesos criticos, incluidos precios, etiquetas y fechas UTC/local.
5. Cambiar la cadena de conexion de la aplicacion a Azure SQL Database.
6. Validar la operacion y conservar el origen en solo lectura durante el periodo de estabilizacion acordado.

## Problemas frecuentes

| Problema | Impacto | Mitigacion |
|---|---|---|
| SHIR sin conectividad privada | Fallan los linked services | Revisar Private Endpoint, DNS privado, rutas y firewall |
| No existe columna de cambios | No hay delta confiable | Usar carga completa o negociar una fuente de cambios aprobada |
| Eliminaciones no detectadas | El destino conserva datos obsoletos | Requiere una fuente existente que registre eliminaciones |
| Reintentos duplican datos | Inconsistencia en destino | Usar una estrategia idempotente de escritura y llaves primarias |
| Objetos de Managed Instance no compatibles | Fallo funcional en Azure SQL Database | Inventariar y adaptar esquema, procedimientos y dependencias |

## Decision

ADF con SHIR sirve para conectividad y copias de datos. Solo usarlo para deltas si la Managed Instance ya expone una señal de cambios que se pueda leer sin alterar la base. De lo contrario, usarlo para carga completa de QA o planear el corte offline mediante una ruta de migracion validada.