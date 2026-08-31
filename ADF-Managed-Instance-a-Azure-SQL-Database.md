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