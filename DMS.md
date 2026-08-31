Para el destino planteado de PAS en Azure SQL Database (serverless o Elastic Pool), Azure DMS se usaría principalmente para una migración offline: carga inicial y corte con una ventana de indisponibilidad. La migración online con sincronización continua desde SQL Server no está soportada hacia Azure SQL Database; para ello el destino tendría que ser Azure SQL Managed Instance o SQL Server en Azure VM.

Ejemplo aplicado a PAS

Crear Azure SQL Database para PAS, preferentemente primero en QA, y migrar esquema, usuarios compatibles y datos históricos.
Aprovisionar Azure DMS con conectividad privada hacia el SQL Server origen y el destino PaaS.
Ejecutar una migración de prueba: DMS valida conectividad, copia esquema/datos y genera reportes de errores.
Conciliar conteos, precios, etiquetas y fechas UTC/local; ejecutar jobs equivalentes y pruebas de carga.
En el corte productivo: detener escrituras de PAS, ejecutar DMS para la carga final, validar conciliación y cambiar la cadena de conexión de la aplicación.
Mantener el origen disponible como respaldo de solo lectura hasta cerrar la ventana de estabilización.
Ventajas

Servicio administrado para mover bases SQL Server hacia Azure.
Reduce trabajo manual de exportar/importar backups o scripts.
Puede detectar incompatibilidades antes del corte mediante evaluaciones y pruebas.
Permite repetir migraciones de QA para medir duración, rendimiento y capacidad requerida.
Mantiene trazabilidad técnica de la operación de carga.
Desventajas y riesgos

. incremental ni el downtime mínimo.
No migra automáticamente SQL Agent Jobs, credenciales, linked servers, configuraciones de servidor ni dependencias externas.
Puede encontrar objetos o características incompatibles con Azure SQL Database que requieren ajuste previo.
No sustituye conciliación funcional: que las filas migren no garantiza que precios, etiquetas ni reglas horarias sean correctos.
No es un mecanismo de rollback: el rollback debe conservar origen, definir el punto de no retorno y tener reversa probada.
Requiere red privada, DNS, firewall/NSG, permisos de origen/destino y capacidad suficiente para la ventana.
Para minimizar indisponibilidad, evaluaría dos caminos:

Mantener Azure SQL Database y combinar DMS para carga inicial con CDC o Change Tracking para cambios incrementales, seguido de un corte final breve.
Usar Azure SQL Managed Instance si el requisito principal es migración online con DMS y mayor compatibilidad con SQL Server.
Referencia: documentación de Azure DMS y la matriz oficial indica que la migración online desde SQL Server no aplica a Azure SQL Database, pero sí a Managed Instance y SQL Server en Azure VM