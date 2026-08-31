# ADF con SHIR vs DMS para migracion de PAS

## Resumen ejecutivo

Azure Data Factory (ADF) con Self-hosted Integration Runtime (SHIR) y Azure Database Migration Service (DMS) no son reemplazos entre si.

- **ADF con SHIR** sirve para mover y sincronizar datos de forma recurrente. Es la opcion para cargas delta controladas por el equipo.
- **DMS** sirve para migrar una base hacia Azure y hacer un corte final. Su sincronizacion online de cambios solo esta disponible en escenarios de origen y destino compatibles.

Para el destino propuesto de **Azure SQL Database**, la estrategia recomendada es: carga inicial con DMS o scripts controlados, sincronizacion de deltas con ADF y SHIR, y un ultimo delta durante la ventana de corte.

## Comparacion

| Aspecto | ADF con SHIR | Azure DMS |
|---|---|---|
| Objetivo | Integracion y movimiento recurrente de datos | Migracion de bases de datos |
| Acceso a origen privado | El SHIR ejecuta la conexion desde una red autorizada | DMS usa la conectividad o el Integration Runtime solicitado en su asistente |
| Carga inicial | Copy Activity configurada por el equipo | Incluida en el flujo de migracion |
| Deltas | Si: watermark, Change Tracking, CDC o consultas | Solo durante migraciones online compatibles, hasta el cutover |
| Uso continuo | Adecuado para procesos programados de largo plazo | No es su fin; termina al completar la migracion |
| Transformaciones | Permite consultas, actividades y orquestacion | Orientado a mover la base, no a transformar datos |
| Control de la logica | Alto; se define que filas leer y como aplicarlas | Bajo; DMS controla el mecanismo de migracion soportado |
| Operacion del corte | El equipo disena la carga final y la reversa | Guiado por el asistente de migracion |

## Deltas con ADF y SHIR

El SHIR que aparece `Running` en Data Factory permite que los linked services lleguen a recursos privados desde la red donde se ejecuta el runtime. Para realizar deltas se requiere una fuente confiable de cambios.

### Alternativas de deteccion de cambios

1. **Watermark:** usar una columna como `UpdatedAt` o `LastModifiedDate`. El pipeline guarda el ultimo valor procesado y consulta solo filas posteriores.
2. **Change Tracking:** SQL Server registra las filas modificadas. Es util si se deben detectar inserciones, actualizaciones y eliminaciones.
3. **CDC:** captura mayor detalle sobre los cambios, a cambio de mas administracion en SQL Server.

Reglas necesarias para cualquier alternativa:

1. Guardar el punto de control solamente despues de que el destino confirme la escritura.
2. Definir como se aplican inserciones, actualizaciones y eliminaciones en el destino.
3. Hacer la escritura idempotente para que un reintento no genere duplicados.
4. Conciliar conteos y datos clave despues de cada carga.

### Prueba de delta

1. Ejecutar una primera corrida para cargar el historico.
2. Insertar o modificar un conjunto pequeno y conocido de filas en el origen.
3. Ejecutar una segunda corrida.
4. En **Monitor > Activity runs**, validar que solo se leyeron y escribieron las filas modificadas.
5. Verificar en el destino que se reflejaron los cambios y que no existen duplicados.

Una conexion exitosa prueba red y credenciales. Esta prueba de dos corridas valida realmente la logica delta.

## DMS y migracion online

DMS es apropiado cuando se busca migrar una base de datos con un corte final administrado. En migraciones online, DMS carga primero el historico y mantiene cambios hasta que se realiza el cutover.

Para SQL Server, este modo online se soporta hacia destinos compatibles, como **Azure SQL Managed Instance** o **SQL Server en Azure Virtual Machines**. Para **Azure SQL Database**, no se debe asumir que DMS realizara una sincronizacion online continua; se debe planear una migracion offline y una carga final durante una ventana de indisponibilidad.

## Network file share en DMS

Cuando en el asistente de DMS se selecciona **Network file share** como ubicacion de backups, se pide configurar un Integration Runtime. Su funcion es permitir que el asistente encuentre y lea archivos `.bak` desde una ruta SMB, por ejemplo:

```text
\\servidor-origen\SQLBackups
```

El entorno que ejecuta ese Integration Runtime debe tener:

- Acceso de red a la ruta SMB.
- Permiso de lectura en el recurso compartido y los archivos de backup.
- Conectividad al SQL Server origen y al destino que solicite el asistente.

No se debe asumir que el SHIR desplegado en App Service puede leer una ruta SMB interna. Validar siempre la ruta desde el equipo o entorno que se use para el Integration Runtime solicitado por DMS.

## Flujo recomendado para PAS hacia Azure SQL Database

1. Ejecutar una evaluacion de compatibilidad y una migracion de prueba con datos de QA.
2. Migrar el historico mediante DMS o scripts controlados.
3. Configurar ADF con el SHIR para aplicar deltas al destino.
4. Validar al menos dos ejecuciones consecutivas con cambios controlados.
5. Durante el corte: detener escrituras en PAS, ejecutar el ultimo delta y conciliar datos, precios y etiquetas.
6. Cambiar la cadena de conexion de la aplicacion al destino PaaS.
7. Mantener el origen en solo lectura durante la estabilizacion y conservar un procedimiento de reversa probado.

## Decision rapida

| Necesidad | Opcion recomendada |
|---|---|
| Mantener sincronizacion delta por dias o semanas | ADF con SHIR |
| Copiar una base y ejecutar un corte final | DMS |
| Migracion online de SQL Server con minimo downtime | DMS hacia Azure SQL Managed Instance o SQL Server en Azure VM |
| Destino Azure SQL Database y necesidad de poco downtime | Carga inicial con DMS + deltas con ADF + corte final |