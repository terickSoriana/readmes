# Migracion de PAS a Azure SQL Database

## Objetivo

Migrar la base de datos PAS desde SQL Server hacia Azure SQL Database con una estrategia controlada, validable y con reversa definida.

---

## Escenario actual

```text
Origen: SQL Server
Destino: Azure SQL Database (PaaS)
Herramienta de migracion: Azure Database Migration Service (DMS)
Modo soportado por DMS: Offline
```

El Integration Runtime ya desplegado en Azure Data Factory confirma que existe conectividad para integrar datos desde recursos privados.

---

## Restriccion clave

La base de datos SQL Server origen **no puede modificarse por ninguna razon**.

Por esta restriccion, no se pueden habilitar ni agregar mecanismos de deteccion de cambios en el origen, por ejemplo:

- Change Tracking.
- Change Data Capture (CDC).
- Triggers.
- Columnas de marca de agua, como `UpdatedAt`.
- Tablas auxiliares dentro de la base origen.

---

## Impacto en la estrategia de deltas

Una carga delta necesita una señal confiable que indique que filas cambiaron. Sin modificar el origen y sin un mecanismo ya existente que exponga esos cambios, ADF no puede identificar solo los registros nuevos, actualizados o eliminados.

Por tanto:

| Alternativa | Resultado |
|---|---|
| ADF con watermark, Change Tracking o CDC | No aplicable si requiere modificar el origen |
| ADF con copia completa recurrente | Posible, pero no es delta; tiene costo, duracion y riesgo operativo mayores |
| DMS hacia Azure SQL Database | Migracion offline; requiere ventana de corte |
| DMS online con deltas | No disponible para el destino Azure SQL Database |

---

## Rol de Azure Data Factory

Azure Data Factory con Self-hosted Integration Runtime es adecuado para integraciones y cargas programadas. En este proyecto puede seguir siendo util para pruebas de conectividad o movimientos controlados de datos.

Sin embargo, no debe plantearse como replica delta si no existe una fuente de cambios aprobada y disponible en el origen.

---

## Rol de Azure DMS

Azure Database Migration Service permite realizar la carga de SQL Server hacia Azure SQL Database en modo offline.

El flujo de DMS incluye:

1. Conexion al SQL Server origen.
2. Conexion al Azure SQL Database destino.
3. Migracion opcional del esquema compatible.
4. Copia de las tablas seleccionadas.
5. Monitoreo por tabla hasta finalizar en `Succeeded`.

Para este destino, DMS no mantiene una sincronizacion continua de cambios antes del corte.

---

## Integration Runtime requerido por DMS

DMS solicita un Self-hosted Integration Runtime propio para ejecutar la copia.

Debe instalarse en un host Windows que tenga conectividad a:

- SQL Server origen por el puerto configurado, normalmente TCP 1433.
- Azure SQL Database destino por TCP 1433.
- Recurso SMB de backups, si se selecciona `Network file share`.

```text
SQL Server origen --> SHIR de DMS --> Azure SQL Database
                         |
                         +--> \\servidor\SQLBackups (si aplica)
```

La clave de registro de DMS es distinta de la clave del SHIR de Azure Data Factory. No se deben intercambiar.

---

## Plan recomendado

### 1. Prueba en QA

1. Instalar y registrar el SHIR solicitado por DMS en un host Windows autorizado.
2. Validar conectividad al SQL Server, Azure SQL Database y file share, si aplica.
3. Ejecutar una migracion de prueba con datos de QA.
4. Medir duracion, rendimiento y errores de compatibilidad.
5. Conciliar conteos, precios, etiquetas y fechas.

### 2. Preparar el corte

1. Definir ventana de indisponibilidad con base en la prueba de QA.
2. Confirmar respaldo y procedimiento de reversa.
3. Validar permisos, firewall, DNS y capacidad de Azure SQL Database.
4. Preparar el cambio de cadena de conexion de la aplicacion.

### 3. Corte productivo

1. Detener escrituras de la aplicacion sobre SQL Server.
2. Confirmar que no existan transacciones pendientes.
3. Ejecutar DMS y monitorear hasta `Succeeded`.
4. Conciliar datos y procesos criticos.
5. Redirigir la aplicacion a Azure SQL Database.
6. Mantener el origen como solo lectura durante la estabilizacion.

---

## Criterios de exito

- Todas las tablas seleccionadas terminan en `Succeeded` en DMS.
- Conteos y datos de negocio criticos coinciden entre origen y destino.
- Precios, etiquetas y conversiones de fecha/hora se validan funcionalmente.
- La aplicacion opera contra Azure SQL Database dentro del SLA acordado.
- Existe evidencia de reversa probada antes del corte productivo.

---

## Decision

Con la prohibicion de modificar SQL Server y con Azure SQL Database como destino, la alternativa viable es una migracion **offline mediante DMS** y una ventana de corte planificada.

La replicacion delta solo debe considerarse si el origen ya tiene una fuente de cambios aprobada que pueda consultarse sin modificar la base, o si cambia la restriccion de arquitectura.