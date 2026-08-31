# DMS: Azure SQL Managed Instance a Azure SQL Database

## Proposito

Esta guia corresponde exclusivamente a una migracion mediante Azure Database Migration Service (DMS).

```text
Origen: Azure SQL Managed Instance
Destino: Azure SQL Database
Objetivo: migracion puntual con corte
```

## Restriccion de soporte

Azure DMS admite migraciones hacia Azure SQL Database en modo **offline**. La documentacion de Microsoft indica que las migraciones online hacia Azure SQL Database no estan disponibles.

Antes de invertir tiempo en configuracion, iniciar una migracion en el portal y confirmar que el asistente permite seleccionar exactamente:

- Source server type: `Azure SQL Managed Instance`.
- Target server type: `Azure SQL Database`.
- Migration mode: `Offline`.

Si esa combinacion no aparece en el portal, no se debe forzar DMS ni configurar un Integration Runtime para ese flujo. Se debe evaluar un mecanismo compatible de exportacion e importacion, o una migracion mediante ADF.

## Cuando usar DMS

Usar DMS si el asistente admite el par origen-destino y el negocio acepta una ventana de indisponibilidad. DMS esta orientado a la carga de una migracion, no a mantener una replicacion permanente.

No usar DMS como solucion de deltas: para Azure SQL Database no hay modo online que mantenga cambios sincronizados hasta el corte.

## Integration Runtime solicitado por DMS

Si el asistente solicita **Configure integration runtime**, ese runtime pertenece a DMS. No es el mismo runtime que se registro en Azure Data Factory.

El SHIR de DMS debe instalarse en un host Windows autorizado que tenga conectividad a:

- Azure SQL Managed Instance origen.
- Azure SQL Database destino por TCP 1433.
- Una ruta SMB de backups, solo si el asistente solicita `Network file share`.

No reutilizar la clave `AUTH_KEY` configurada para el contenedor de ADF. En **DMS > Settings > Integration runtime**, seleccionar **Configure integration runtime**, descargar el instalador y registrarlo con la clave entregada por DMS. El nodo debe aparecer como `Online` antes de continuar.

## Flujo de migracion

1. Validar que el asistente DMS admita `Azure SQL Managed Instance -> Azure SQL Database`.
2. Ejecutar una evaluacion de compatibilidad: Azure SQL Database tiene menos funcionalidades de instancia que Managed Instance.
3. Crear el Azure SQL Database destino y configurar su red, firewall o Private Endpoint.
4. Configurar el SHIR de DMS solo si el asistente lo solicita.
5. Ejecutar una prueba completa en QA y registrar duracion, errores y objetos incompatibles.
6. Conciliar conteos, llaves, procedimientos, precios, etiquetas y fechas.
7. Programar una ventana de corte basada en la duracion de QA.
8. Detener escrituras en el origen, ejecutar DMS y esperar estado `Succeeded`.
9. Validar el destino y cambiar la conexion de la aplicacion.
10. Conservar el origen en modo solo lectura durante la estabilizacion y tener una reversa aprobada.


## 1. Crear una VM Windows temporal para el SHIR

Antes de crear recursos, validar que el asistente de DMS permita exactamente este par:

```text
Source server type: Azure SQL Managed Instance
Target server type: Azure SQL Database
Migration mode: Offline
```

Si ese par no aparece, detener esta configuracion: no se debe crear el SHIR hasta validar un metodo de migracion compatible.

### Red de la VM

1. En Azure Portal, abrir la VNet donde esta la Managed Instance.
2. Crear una subnet nueva para la VM, por ejemplo `snet-dms-ir`.
3. No usar la subnet delegada de la Managed Instance; esa subnet es exclusiva para recursos de Managed Instance.
4. Si la VM se crea en otra VNet, configurar VNet peering, rutas y NSG para permitir conectividad hacia la Managed Instance.

### Crear la VM

1. En Azure Portal, seleccionar **Virtual machines > Create > Azure virtual machine**.
2. Elegir imagen `Windows Server 2022 Datacenter`.
3. Seleccionar un tamano con al menos 4 vCPU, 8 GB de RAM y 80 GB disponibles en disco.
4. En **Networking**, seleccionar la subnet `snet-dms-ir`.
5. Usar Azure Bastion o RDP restringido a una IP de administracion. No exponer RDP a Internet sin restriccion.
6. Crear la VM y conectarse como administrador.

### Reglas de conectividad

Permitir desde la VM:

- Conexion a la Managed Instance por TCP 1433, usando su FQDN privado.
- Conexion a Azure SQL Database por TCP 1433 o al Private Endpoint configurado.
- Salida HTTPS por TCP 443 para que el Integration Runtime se registre y se comunique con Azure.

## 2. Usar un servidor Windows existente para el SHIR

Esta alternativa evita crear una VM temporal, pero el servidor existente se convierte en el equipo de computo de la migracion. Usarlo solo despues de confirmar que DMS admite el par `Azure SQL Managed Instance -> Azure SQL Database` en modo `Offline`.

### Requisitos del servidor

- Windows Server 2016, 2019, 2022 o 2025 de 64 bits.
- .NET Framework 4.7.2 o superior y permisos de administrador local para instalar el runtime.
- Como referencia: 4 vCPU, 8 GB de RAM y 80 GB libres en disco.
- No usar un controlador de dominio ni un servidor con carga critica durante la ventana de migracion.
- El servidor debe permanecer encendido mientras DMS ejecute la migracion.
- Confirmar que no tenga ya instalado otro Self-hosted Integration Runtime; solo puede haber una instancia por equipo.

### Validar red antes de instalar

Obtener el FQDN de la Managed Instance en **Azure Portal > Managed Instance > Overview > Host**. Desde el servidor Windows, ejecutar:

```powershell
Resolve-DnsName <fqdn-managed-instance>
Test-NetConnection <fqdn-managed-instance> -Port 1433
Test-NetConnection <servidor-destino>.database.windows.net -Port 1433
```

Los dos comandos `Test-NetConnection` deben indicar `TcpTestSucceeded : True`. Si el destino usa Private Endpoint, el FQDN debe resolver a la direccion IP privada esperada. Tambien se requiere salida HTTPS por TCP 443 hacia Azure; configurar el proxy corporativo si aplica.

### Instalar y registrar

1. En **DMS > Settings > Integration runtime**, seleccionar **Configure integration runtime**.
2. Descargar el instalador en el servidor Windows y ejecutarlo como administrador.
3. Registrar el runtime con la clave generada por DMS. No usar la clave `AUTH_KEY` del runtime de ADF.
4. Confirmar que el nodo aparezca como `Online` en DMS antes de continuar con el asistente de migracion.
5. Ejecutar una prueba de QA y monitorear CPU, memoria, disco y conectividad del servidor durante toda la copia.


## Problemas frecuentes

| Problema | Impacto | Mitigacion |
|---|---|---|
| El par origen-destino no aparece en DMS | No existe ruta soportada por ese asistente | Detener configuracion y evaluar ADF o exportacion/importacion compatible |
| Funcionalidad exclusiva de Managed Instance | Objetos o codigo no compatibles con Azure SQL Database | Corregir antes del corte y probar en QA |
| Falta de conectividad privada | DMS o SHIR no conecta al origen o destino | Revisar DNS, rutas, NSG, firewall y Private Endpoints |
| Ventana insuficiente | La migracion no termina a tiempo | Medir QA, aumentar capacidad destino o redefinir el corte |
| Expectativa de deltas online | Riesgo de perdida de cambios | Planear corte offline o usar una estrategia ADF aprobada |

## Resultado esperado

DMS termina todas las tablas seleccionadas en `Succeeded`, la conciliacion funcional es satisfactoria y la aplicacion puede cambiar a Azure SQL Database dentro de la ventana acordada.