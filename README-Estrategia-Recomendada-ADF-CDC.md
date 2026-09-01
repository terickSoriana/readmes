# Decision de migracion: ADF con CDC

## Decision ejecutiva

Se propone migrar desde **Azure SQL Managed Instance** hacia **Azure SQL Database** mediante una carga inicial con **Azure Data Factory (ADF)** y sincronizacion incremental con **Change Data Capture (CDC)**, consumida por ADF hasta el corte productivo.

```text
Origen: Azure SQL Managed Instance
Destino: Azure SQL Database
Estrategia elegida: ADF para carga inicial + CDC para deltas + ADF para aplicarlos
Objetivo: reducir la indisponibilidad y capturar INSERT, UPDATE y DELETE
Condicion: aprobar formalmente la habilitacion de CDC en el origen
```

La alternativa queda sujeta a una prueba completa en QA y a la aprobacion del administrador de la Managed Instance. CDC modifica el origen: crea objetos de captura y requiere capacidad de log, almacenamiento y monitoreo.

## Problema que resuelve

Una carga completa toma tiempo. Mientras ADF copia los datos historicos, la aplicacion puede seguir generando altas, cambios y eliminaciones en la Managed Instance. Si esos cambios no se capturan, el destino queda desactualizado al terminar la carga y el corte exige una ventana larga.

CDC registra esos cambios en el origen. ADF los lee por intervalos de LSN y los aplica en Azure SQL Database. De esta forma, la carga completa se realiza antes del corte y la ventana productiva se limita al ultimo delta, conciliacion y cambio de conexion.

## Arquitectura propuesta

```text
Azure SQL Managed Instance
  - Tablas de aplicacion
  - CDC habilitado por tabla
            |
            | SHIR de ADF por red privada
            v
Azure Data Factory
  - Pipeline de carga inicial
  - Pipeline de deltas CDC
  - Programacion, alertas y reintentos
            |
            v
Azure SQL Database
  - stg: datos de aterrizaje
  - ctl: LSN y estado de corridas
  - auditoria: cambios y resultados
  - dbo: tablas operativas
```

ADF transporta y orquesta. La logica de aplicacion de cambios debe vivir en procedimientos almacenados idempotentes en Azure SQL Database, no en consultas manuales de cada corrida.

## Ventajas de la estrategia elegida

1. **Baja indisponibilidad.** La mayor parte del volumen se mueve antes del corte; durante la ventana solo se procesa el delta final.
2. **Deltas completos.** CDC detecta `INSERT`, `UPDATE` y `DELETE` fisicos. Un watermark basado en fecha o `rowversion` normalmente no puede detectar eliminaciones.
3. **Sin huecos durante la carga inicial.** CDC se activa antes de iniciar la copia completa, por lo que registra los cambios que ocurran mientras esta se ejecuta.
4. **Menor carga recurrente.** Despues de la carga inicial, se consultan cambios y no todas las filas de cada tabla.
5. **Trazabilidad.** Los limites LSN permiten registrar que intervalo fue aplicado, reprocesar una corrida fallida y auditar filas procesadas.
6. **Uso de servicios ya alineados con Azure.** ADF, SHIR, Managed Instance y Azure SQL Database se integran con controles de red privada, credenciales administradas y monitoreo de Azure.
7. **Separacion de responsabilidades.** El origen captura cambios; ADF orquesta; el destino aplica, controla y audita. Esto reduce el riesgo de scripts manuales durante el corte.

## Desventajas y controles requeridos

| Riesgo o desventaja | Control requerido |
|---|---|
| CDC modifica la Managed Instance | Aprobacion formal, respaldo validado y ejecucion inicial en QA |
| Uso adicional de log y almacenamiento | Medir crecimiento, latencia de captura y capacidad antes de produccion |
| ADF puede atrasarse frente a la retencion de CDC | Alertar por retraso y definir una accion: reprocesar, reconciliar o recargar |
| Cambios de esquema no se propagan por si solos | Congelar cambios de esquema durante la migracion o incluirlos en un proceso versionado y probado |
| La reversa no es automatica despues de nuevas escrituras en destino | Definir una ventana de estabilizacion, escritura dual temporal o reproceso de operaciones |
| Las tablas sin llave primaria tienen limitaciones para cambios netos | Inventariar llaves antes de habilitar CDC y tratar excepciones por tabla |

## Secuencia de implementacion

### Fase 0: aprobacion y preparacion

1. Confirmar que Azure SQL Database tiene capacidad, conectividad privada y compatibilidad funcional para la aplicacion.
2. Inventariar tablas, llaves primarias, volumen, dependencias y prioridad de cada tabla.
3. Definir retencion de CDC, frecuencia de los deltas y el desfase maximo tolerado.
4. Crear en Azure SQL Database los esquemas `stg`, `ctl` y `auditoria`, las tablas operativas y procedimientos idempotentes.
5. Validar el SHIR de ADF contra ambos extremos por TCP 1433 y salida HTTPS requerida por Azure.
6. Acordar criterios de aceptacion: conteos, llaves, precios, etiquetas, fechas UTC/local y pruebas funcionales.

### Fase 1: habilitar CDC antes de copiar

CDC debe activarse antes de la carga inicial. Hacerlo despues crea un intervalo de cambios no capturados.

En la base origen, con una cuenta autorizada:

```sql
USE BaseOrigen;
GO

EXEC sys.sp_cdc_enable_db;
GO

EXEC sys.sp_cdc_enable_table
    @source_schema = N'dbo',
    @source_name = N'Clientes',
    @role_name = NULL,
    @supports_net_changes = 1;
GO
```

Repetir `sys.sp_cdc_enable_table` por cada tabla aprobada. Antes de continuar, comprobar que la tabla quedo habilitada y guardar su `capture_instance`.

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

### Fase 2: registrar LSN inicial y ejecutar carga completa

1. Guardar en `ctl` el LSN inicial de cada tabla CDC.
2. Ejecutar ADF para copiar el historico completo a `stg.<Tabla>`.
3. Ejecutar el procedimiento del destino para cargar staging a la tabla operativa.
4. Conciliar conteos, llaves y valores criticos.
5. No avanzar el LSN confirmado hasta que la carga y la conciliacion sean correctas.

Ejemplo para consultar el punto inicial de una instancia de captura:

```sql
SELECT sys.fn_cdc_get_min_lsn(N'dbo_Clientes') AS LsnInicial;
```

Los cambios generados mientras ocurre la carga completa permanecen disponibles en CDC y se aplican en el primer delta.

### Fase 3: ejecutar deltas recurrentes

Cada pipeline incremental debe realizar esta secuencia por tabla:

1. Leer de `ctl` el LSN confirmado anterior.
2. Consultar un LSN superior al inicio de la corrida para fijar el intervalo.
3. Leer los cambios CDC entre ambos LSN y llevarlos a staging.
4. Aplicar los cambios mediante un procedimiento idempotente.
5. Registrar filas leidas, insertadas, actualizadas, eliminadas, hora de inicio/fin y resultado.
6. Actualizar el LSN confirmado solo cuando todas las actividades hayan terminado correctamente.

El procedimiento en destino debe interpretar `__$operation` y aplicar las eliminaciones. Nunca debe actualizar el LSN de control antes de confirmar que los cambios llegaron correctamente a las tablas operativas.

La frecuencia inicial debe basarse en QA; como referencia, ejecutar cada 15, 30 o 60 minutos segun volumen, duracion y desfase aceptable.

### Fase 4: pruebas obligatorias en QA

1. Hacer una carga inicial con volumen representativo y medir tiempo, CPU, memoria, log y almacenamiento.
2. Generar inserciones, actualizaciones y eliminaciones controladas en el origen.
3. Ejecutar al menos dos deltas consecutivos y validar que no haya duplicados ni cambios omitidos.
4. Forzar un fallo despues de aterrizar staging y comprobar que el reintento sea seguro.
5. Simular cambios mientras se ejecuta una carga completa y comprobar que el primer delta los incluya.
6. Validar el atraso maximo frente a la retencion configurada de CDC.
7. Medir el ultimo delta y la conciliacion final para estimar la ventana de corte.

### Fase 5: corte productivo

1. Confirmar respaldo, plan de reversa, lista de contactos y criterios de aceptacion.
2. Detener escrituras hacia la Managed Instance y esperar las transacciones pendientes.
3. Esperar que CDC capture los ultimos cambios y ejecutar el delta final.
4. Confirmar que el LSN final del origen coincide con el LSN aplicado en destino.
5. Conciliar tablas y procesos criticos.
6. Cambiar la cadena de conexion de la aplicacion hacia Azure SQL Database.
7. Monitorear la aplicacion y el destino durante el periodo de estabilizacion.

## Reversa

La reversa es simple solo mientras Azure SQL Database no reciba escrituras de negocio:

1. Detener la aplicacion.
2. Restaurar la cadena de conexion hacia Azure SQL Managed Instance.
3. Validar la operacion en el origen.

Despues de que el destino recibe escrituras, no se debe volver al origen sin un plan para esos datos. Elegir y probar una de estas medidas antes del corte:

- Mantener el destino en solo lectura hasta completar validaciones.
- Implementar escritura dual temporal desde la aplicacion.
- Registrar operaciones nuevas y reprocesarlas en el origen bajo un procedimiento aprobado.

CDC por si solo opera de origen a destino; no realiza sincronizacion inversa ni resuelve conflictos bidireccionales.

## ADF con CDC vs DMS offline

La eleccion no es entre dos herramientas equivalentes. DMS offline esta orientado a mover una base en una unica ejecucion de migracion; ADF con CDC esta orientado a cargar el historico y mantener el destino actualizado mediante corridas incrementales controladas.

| Requisito para esta migracion | ADF con CDC | DMS offline | Impacto de DMS en este escenario |
|---|---|---|---|
| Cargar el historico antes del corte | Si, con pipeline de carga inicial | Si, en la ejecucion de migracion | Ambos pueden mover los datos iniciales |
| Mantener deltas antes del corte | Si, por LSN de CDC | No | El destino queda desactualizado si la aplicacion continua escribiendo mientras se prepara el corte |
| Capturar `INSERT`, `UPDATE` y `DELETE` | Si, para tablas CDC habilitadas | Solo como estado final durante la carga offline | DMS no entrega una secuencia continua de cambios para sincronizar el destino |
| Reducir la ventana de indisponibilidad | Si; la ventana contiene el ultimo delta y la conciliacion | No necesariamente; incluye la copia productiva completa | La duracion depende del volumen total y puede exceder la ventana disponible |
| Ejecutar deltas recurrentes y auditables | Si, con LSN, tabla `ctl`, staging y resultados por corrida | No | No hay forma de validar el atraso ni reprocesar solo un intervalo de cambios |
| Reintentar sin perder cambios | Si, al conservar el LSN hasta confirmar la aplicacion | Requiere reiniciar o repetir la migracion segun el estado | Mayor riesgo operativo y menor granularidad de recuperacion |
| Reconciliar mientras el origen sigue activo | Si, durante las corridas CDC | Limitado; la conciliacion final requiere un origen estable | La aplicacion debe detenerse antes de asegurar consistencia final |
| Reversa antes de aceptar escrituras en destino | Si, revirtiendo la conexion | Si, revirtiendo la conexion | Ambos permiten volver mientras el destino no reciba operaciones de negocio |
| Reversa despues de escribir en destino | Requiere escritura dual o reproceso | Requiere escritura dual o reproceso | Ninguna alternativa la resuelve por si sola |
| Cambios en la Managed Instance | Si, CDC debe habilitarse | Menores para una migracion puntual | Esta es la principal ventaja de DMS, pero se intercambia por una ventana mas larga |

### Por que DMS puede no cumplir los requisitos

DMS puede ser correcto para un corte offline, pero no cumple por si solo los requisitos de esta estrategia cuando se necesita mover el historico con anticipacion, mantener deltas y reducir el tiempo sin servicio.

1. **No conserva el destino actualizado antes del corte.** DMS offline no actua como replicacion continua. Si se hiciera una carga de prueba dias antes, las escrituras posteriores de la aplicacion permanecerian solo en la Managed Instance.
2. **Exige detener escrituras durante toda la copia productiva.** Para lograr consistencia, la aplicacion se detiene antes de iniciar la migracion productiva y no puede volver a escribir hasta que finalice la copia, conciliacion y cambio de conexion.
3. **La ventana depende del volumen total.** En ADF con CDC, el volumen principal se mueve antes y el corte procesa el acumulado desde el ultimo LSN. Con DMS offline, el tiempo de mover todas las tablas forma parte de la indisponibilidad.
4. **No aporta deltas ni una cola de cambios.** No hay un equivalente al LSN para aplicar periodos cortos, verificar atraso o reintentar exactamente el intervalo que fallo.
5. **No mejora la reversa despues del cambio.** DMS no replica de regreso las escrituras nuevas de Azure SQL Database hacia Managed Instance. Si se necesita volver despues de recibir operaciones en destino, se requiere la misma disciplina adicional que con ADF: solo lectura, escritura dual o reproceso controlado.
6. **No elimina el trabajo de compatibilidad.** Aunque DMS ayude con la migracion, siguen siendo necesarios el analisis de objetos no compatibles, las pruebas funcionales, la conciliacion y los cambios de aplicacion necesarios para Azure SQL Database.

### Cuando DMS si seria una mejor eleccion

DMS offline es preferible si todas estas condiciones son verdaderas:

1. El negocio acepta una ventana de mantenimiento que cubra la copia completa, conciliacion y cambio de conexion.
2. No se requiere mantener un destino actualizado antes de la ventana.
3. No esta permitido habilitar CDC ni realizar cambios operativos en el origen.
4. El volumen de datos y la duracion medida en QA caben con margen en la ventana aprobada.
5. La aplicacion puede permanecer detenida hasta que termine la migracion y las validaciones.

## Por que no se seleccionaron las demas opciones

### DMS offline

No se selecciona como estrategia principal porque no mantiene deltas continuos hacia Azure SQL Database. Obliga a detener escrituras durante la migracion productiva completa; la ventana dependeria del tiempo total de copiar el volumen, no solo del ultimo delta. Puede ser un plan alterno si el negocio acepta una indisponibilidad amplia y no aprueba CDC.

### ADF con carga completa recurrente

No se selecciona porque leer toda la tabla en cada corrida incrementa costo, duracion y carga en la Managed Instance. Aunque permite comparar staging y detectar diferencias, no es una replicacion incremental real y se vuelve menos viable a medida que el volumen crece.

### ADF con watermark existente

No se selecciona como estrategia general porque depende de que todas las tablas tengan una fecha de modificacion o `rowversion` confiable. Ademas, no detecta eliminaciones fisicas sin una marca de borrado logico ya existente. Puede usarse como excepcion en tablas donde la señal ya exista y haya sido validada.

### Escritura dual desde la aplicacion

No se selecciona como ruta principal porque requiere cambios relevantes en la aplicacion: idempotencia, reintentos, orden de eventos, tratamiento de fallos parciales y reconciliacion. Es la mejor opcion complementaria si la reversa sin perdida despues del corte es un requisito estricto.

### Replicacion transaccional publisher-subscriber

No se selecciona inicialmente porque requiere configurar publicacion, distribucion y suscripcion, ademas de validar el soporte exacto, permisos, red, retencion y comportamiento de esquema para este par de servicios. Tambien modifica el origen, opera principalmente en un sentido y no entrega reversa automatica. Se puede evaluar como alternativa si el equipo de base de datos ya opera replicacion transaccional y confirma su compatibilidad en QA.

### Herramientas de terceros con CDC

No se seleccionan porque agregan licencia, infraestructura, soporte y dependencias de proveedor. Muchas igualmente requieren acceso a logs, CDC o cambios de permisos en el origen. Solo se justifican si ADF no cumple la latencia, el volumen o los objetivos operativos demostrados en QA.

### BACPAC o SqlPackage

No se seleccionan porque son mecanismos puntuales de exportacion e importacion; no mantienen deltas. La ventana productiva incluiria la exportacion/importacion completa y el riesgo de incompatibilidades de esquema es mayor. Son utiles para pruebas, bases pequenas o un corte con indisponibilidad aceptada.

## Criterios de aprobacion para produccion

No programar el corte hasta que todos los puntos sean verdaderos:

- CDC esta habilitado y monitoreado en todas las tablas aprobadas.
- La carga inicial y al menos dos deltas consecutivos terminaron correctamente en QA.
- Se validaron inserciones, actualizaciones, eliminaciones, reintentos y cambios durante la carga inicial.
- El atraso observado de ADF es menor que la retencion de CDC con margen operativo.
- Los conteos, llaves y validaciones funcionales coinciden.
- La duracion del delta final cabe dentro de la ventana aprobada.
- El plan de reversa tiene responsables, procedimiento probado y criterio de decision.
- El cambio de conexion de la aplicacion fue probado en QA.

## Responsables sugeridos para la sesion

| Rol | Decision o evidencia requerida |
|---|---|
| Administrador de base de datos | Autoriza CDC, retencion, permisos, capacidad y respaldo |
| Equipo de aplicacion | Valida compatibilidad, cadena de conexion, pruebas funcionales y reversa |
| Equipo de datos | Construye pipelines ADF, staging, control de LSN, auditoria y alertas |
| Infraestructura y red | Confirma SHIR, DNS, rutas, NSG, firewall y Private Endpoints |
| Negocio | Define desfase aceptable, ventana de corte y criterios de aceptacion |

## Conclusion

ADF con CDC se selecciona porque equilibra baja indisponibilidad, captura completa de cambios y control operativo con servicios de Azure. La decisión solo es válida si se autoriza modificar la Managed Instance y la prueba de QA demuestra que el origen soporta la carga adicional. Si CDC no obtiene aprobación, la alternativa conservadora es ADF con watermark existente o comparación completa en staging, aceptando sus límites de volumen y eliminaciones.