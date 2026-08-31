# Seguimiento de Migracion PAS a PaaS

## 1. Objetivo y alcance

Migrar la base de datos PAS a una plataforma PaaS escalable y evaluar la integracion con la base de datos de Fenicia Services. La solucion debe soportar los procesos recurrentes de precios y etiquetas sin afectar la operacion de las tiendas.

## 2. Demanda y capacidad

| Concepto | Dato conocido | Pendiente de definir |
|---|---:|---|
| Cambios de precio | 500 a 1,000 por ciclo | Frecuencia, ventana de procesamiento y picos |
| Tiendas | 800 | SLA de publicacion por tienda |
| Eventos potenciales | Hasta 800,000 por ciclo | Tamano de lote e idempotencia |

Para una carga variable, evaluar:

- **Azure SQL Database serverless:** adecuado si se tolera el tiempo de reanudacion tras periodos sin uso.
- **Elastic Pool:** adecuado si PAS y otras bases mantienen administracion separada, pero presentan consumo variable.

La consolidacion de PAS y Fenicia Services no debe decidirse solo por costo. Requiere validar compatibilidad de modelo de datos, ciclo de vida, seguridad, rendimiento, respaldos y ventanas de mantenimiento.

## 3. Zona horaria y datos de etiquetas

Las bases PAS operan en GMT y la informacion se genera un dia antes. Para datos de etiquetas se identifica un desfase potencial de seis horas respecto a la hora local.

Acciones requeridas:

1. Almacenar instantes de tiempo en UTC.
2. Convertir a hora local solo en la presentacion o en reglas de negocio definidas.
3. Probar cambios de precio y etiquetas cercanos a medianoche, incluyendo horario de verano cuando aplique.
4. Definir el corte operativo y la hora limite de generacion de etiquetas.

## 4. Migracion en dos fases

| Fase | Actividades | Criterio de salida |
|---|---|---|
| 1. Carga inicial y validacion | Migrar historico, validar esquema, volumen, integridad y rendimiento en QA | Conciliacion aprobada y pruebas funcionales satisfactorias |
| 2. Sincronizacion y corte | Aplicar cambios incrementales, ejecutar conciliacion final y cambiar la aplicacion a PaaS | Operacion estable dentro de la ventana acordada |

Azure Database Migration Service puede apoyar la migracion de datos, pero no sustituye el plan de reversa de negocio.

## 5. Rollback y continuidad

Definir antes del corte:

- Punto de no retorno y responsables de aprobarlo.
- Respaldo de origen y destino, con restauracion probada.
- Origen en solo lectura durante el corte o doble escritura temporal, segun el diseno.
- Conciliacion de registros, precios y etiquetas.
- Procedimiento de reversa, ventana de decision y responsables operativos.

Para sincronizacion incremental, evaluar CDC o Change Tracking. Azure SQL Data Sync puede servir en casos limitados, pero no debe ser el mecanismo principal de rollback por su latencia, manejo de conflictos y complejidad adicional.

## 6. Jobs recurrentes y orquestacion

No se deben migrar los SQL Agent Jobs de forma directa sin inventario y clasificacion. Para cada job se requiere registrar frecuencia, dependencias, credenciales, duracion, criticidad, ventanas de ejecucion, reintentos y mecanismo de alerta.

| Tipo de proceso | Alternativa recomendada | Consideraciones |
|---|---|---|
| Logica transaccional de precios | Procedimientos almacenados en Azure SQL | Debe mantener idempotencia, auditoria y control de concurrencia |
| Procesos SQL recurrentes | Azure SQL Elastic Jobs | Alternativa directa para ejecuciones centradas en base de datos |
| Integraciones, APIs, archivos y notificaciones | n8n, Azure Functions, Logic Apps o Data Factory | El orquestador debe invocar la logica SQL, no sustituirla |
| Flujos criticos con SLA, dependencias complejas u operacion 24x7 | Control-M | Mantener o integrar si es el estandar operativo |

### Analisis requerido para Control-M

Para decidir si los jobs recurrentes se mantienen en Control-M, analizar:

1. Calendarios, frecuencia, ventanas de ejecucion y dependencia entre jobs.
2. SLA, criticidad, reintentos, alertamiento y escalamiento operativo.
3. Acceso desde el servidor central de QA hacia PaaS: rutas, DNS, puertos, firewall, cuentas de servicio y secretos.
4. Capacidad de evitar ejecuciones duplicadas y de recuperar ejecuciones fallidas.
5. Trazabilidad de ejecuciones, retencion de bitacoras y evidencia para auditoria.
6. Pruebas de conectividad y ejecucion extremo a extremo antes de produccion.

El servidor central de QA debe contar con conectividad privada aprobada hacia PaaS. Se deben documentar y autorizar los flujos necesarios a traves de firewall, NSG y DNS privado, sin exponer la base de datos a Internet.

### Inventario inicial de jobs

| Job | Frecuencia | Dependencias | Duracion | Criticidad | Alternativa preliminar |
|---|---|---|---|---|---|
| Actualizacion de precios | Por confirmar | PAS, tiendas | Por confirmar | Alta | Control-M o Elastic Jobs + procedimiento almacenado |
| Envio de etiquetas | Diario, por confirmar | Zona horaria, precios | Por confirmar | Alta | Control-M o n8n HA como orquestador |
| Limpieza y mantenimiento | Semanal, por confirmar | Base de datos | Por confirmar | Media | Azure SQL Elastic Jobs |

## 7. POC de viabilidad en Azure

La POC debe ejecutarse primero con una copia anonimizada o aprobada de QA. Su objetivo es demostrar que los procedimientos almacenados (SP) criticos de PAS pueden compilarse, ejecutarse y cumplir su ventana operativa en Azure SQL Database. No debe conectarse produccion directamente al portal ni exponer la base de datos a Internet.

### Informacion que se debe solicitar

1. Nombre, version y edicion de cada SQL Server origen; nombre de la base PAS y ambiente correspondiente.
2. Respaldo o copia de QA, tamano de la base, crecimiento diario y datos de volumen de las tablas principales.
3. Script de esquema y de todos los SP, funciones, vistas, tipos definidos por usuario, triggers y tablas involucradas.
4. Inventario priorizado de SP: actualizacion de precios, envio de etiquetas, consultas de tiendas, limpieza y cualquier proceso con SLA.
5. Para cada SP critico: parametros de entrada, resultado esperado, volumen representativo, duracion actual, frecuencia, dependencias externas y datos de prueba.
6. Dependencias de instancia: SQL Agent Jobs, linked servers, credenciales, Database Mail, rutas de archivos, comandos del sistema, CLR, Service Broker y accesos a otras bases.
7. Acceso de red aprobado desde una VM o red de QA hacia Azure: VPN o ExpressRoute, DNS privado, reglas de firewall/NSG y cuenta con permisos de lectura en origen y de administracion en destino.

### Ambiente minimo

1. Crear un grupo de recursos de POC y una Azure SQL Database de QA. Elegir inicialmente aprovisionada o Elastic Pool para medir rendimiento de manera estable; evaluar serverless despues si los periodos de reanudacion son aceptables.
2. Configurar un Private Endpoint y zona DNS privada. El acceso debe probarse desde la red de QA; solo como prueba temporal y autorizada se podria habilitar una regla de firewall publica restringida a una IP de administracion.
3. Crear un usuario administrador de POC y usuarios de aplicacion con privilegio minimo. No reutilizar credenciales de produccion.
4. Aprovisionar Azure DMS con acceso privado al SQL Server de QA y a Azure SQL Database si se usara para la carga de prueba.

### Ejecucion de la POC

1. Ejecutar una evaluacion de compatibilidad antes de migrar. Clasificar cada hallazgo como bloqueante, corregible o no aplicable; los bloqueantes determinan si debe ajustarse el codigo o considerarse Azure SQL Managed Instance.
2. Migrar esquema y una muestra representativa de datos. Usar Azure DMS para una migracion offline o scripts controlados; registrar duracion, errores y objetos no migrados.
3. Desde Azure Portal, abrir la base de datos y usar **Query editor** con el usuario de POC para ejecutar consultas de validacion y SP pequenos. Para pruebas con archivos, lotes grandes o conectividad privada, usar Azure Data Studio o SSMS desde una VM de QA conectada a la red privada.
4. Ejecutar cada SP critico con sus parametros y volumen representativo. Comparar resultado, conteo de filas, precios, etiquetas, fechas UTC/local y efectos idempotentes contra el origen de QA.
5. Simular un ciclo completo de precios para 800 tiendas y el envio de etiquetas. Medir duracion total, consumo de CPU, DTU o vCore, bloqueos, tiempos de espera y errores de reintento.
6. Probar el orquestador seleccionado para los jobs: invocar el SP desde Control-M, Elastic Jobs u otro componente y verificar calendario, reintentos, alerta, bitacora y proteccion contra ejecuciones duplicadas.
7. Repetir la carga para estimar la ventana de corte. Para Azure SQL Database, probar el proceso equivalente a detener escrituras, cargar el delta final, conciliar y cambiar la conexion; no asumir sincronizacion online nativa con DMS.

### Matriz de resultados y decision

| Prueba | Evidencia requerida | Criterio de aceptacion |
|---|---|---|
| Compatibilidad de esquema y SP | Reporte de evaluacion y lista de correcciones | Sin bloqueantes abiertos o con plan aprobado y probado |
| Conectividad privada | Conexion desde VM de QA mediante DNS privado | Sin acceso publico permanente ni fallas de autenticacion |
| Exactitud funcional | Comparacion de conteos, precios, etiquetas y fechas | Resultados iguales al origen para los casos acordados |
| Rendimiento | Duracion y metricas del ciclo representativo | Cumple el SLA y deja margen operativo acordado |
| Orquestacion | Bitacora de una ejecucion programada y una falla recuperada | Reintento, alerta e idempotencia verificados |
| Corte y reversa | Cronometro de carga final, conciliacion y retorno a origen | Cabe en la ventana acordada y la reversa es ejecutable |

La POC es exitosa para Azure SQL Database si los SP criticos no requieren funciones exclusivas de instancia, cumplen los criterios anteriores y la carga final cabe en la ventana de indisponibilidad. Si hay dependencia fuerte de SQL Agent, linked servers, funcionalidades de servidor o se exige migracion online con DMS, se debe evaluar Azure SQL Managed Instance como alternativa de destino.

## 8. Proximos pasos

1. Confirmar volumen, picos, SLA y ventanas operativas para precios y etiquetas.
2. Inventariar los jobs actuales y clasificarlos con la tabla anterior.
3. Definir si PAS y Fenicia Services se mantienen separadas o se consolidan, con base en criterios tecnicos y operativos.
4. Disenar la conectividad privada de QA y Control-M hacia PaaS, incluyendo reglas de firewall, DNS y cuentas de servicio.
5. Definir la estrategia incremental, el plan de corte y el rollback probado.
6. Ejecutar prueba integral en QA: carga, sincronizacion, jobs recurrentes, conciliacion y reversa.
