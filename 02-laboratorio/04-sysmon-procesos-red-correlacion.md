# Guía práctica — Sysmon: procesos, red y correlación de eventos

**Fecha de esta etapa:** 2026-08-19  
**Laboratorio:** `bluelab.test`  
**Controlador de dominio:** `LAB-DC01`  
**Cliente Windows 11:** `LAB-CLIENT01`  
**Objetivo:** ampliar la telemetría nativa de Windows con Sysmon, observar creación/finalización de procesos y conexiones de red, y correlacionar eventos por proceso para investigar actividad de endpoint con criterio Blue Team/SOC.

---

## 1. Punto de partida

La etapa anterior dejó operativa la auditoría nativa de Windows en `LAB-CLIENT01`:

```text
4624 → inicio de sesión correcto
4625 → inicio de sesión fallido
4672 → privilegios especiales asignados
4688 → creación de procesos
      + proceso padre
      + línea de comandos
      + nivel de elevación
```

La GPO usada fue:

```text
GPO-WKS-AUDITORIA-BLUE-TEAM
```

vinculada a:

```text
BLUELAB
└── EQUIPOS
    └── WORKSTATIONS
        └── LAB-CLIENT01
```

La lógica de trabajo se mantiene:

```text
configurar telemetría
→ verificar que realmente funciona
→ generar actividad controlada
→ localizar el evento
→ leer campos relevantes
→ correlacionar
→ interpretar contexto
→ documentar conclusión
```

---

## 2. Por qué incorporar Sysmon

El evento nativo `4688` permite conocer la creación de procesos, pero Sysmon entrega contexto adicional útil para hunting e investigación.

Ejemplo de campos observados en Sysmon Event ID 1:

```text
ProcessGuid
ProcessId
Image
CommandLine
User
LogonGuid
Hashes
ParentProcessGuid
ParentProcessId
ParentImage
ParentCommandLine
```

### Diferencia práctica

```text
Windows 4688
→ proceso
→ usuario
→ proceso padre
→ línea de comandos
→ elevación

Sysmon Event 1
→ proceso
→ usuario
→ línea de comandos
→ proceso padre
→ hashes
→ ProcessGuid
→ ParentProcessGuid
```

El objetivo no es reemplazar la auditoría nativa, sino complementar la visibilidad del endpoint.

---

## 3. Comprobación previa

Antes de instalar Sysmon se verificó que no existiera un servicio Sysmon previo:

```powershell
Get-Service sysmon*
```

No devolvió resultados.

### Por qué se hizo

Antes de instalar un componente de telemetría se debe comprobar si ya existe una instalación, para no crear configuraciones duplicadas o servicios en conflicto.

---

## 4. Habilitación de Sysmon integrado

En `LAB-CLIENT01` se utilizó PowerShell.

Primer intento:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Sysmon
```

Resultado:

```text
La operación solicitada requiere elevación.
```

La consola no estaba ejecutada como administrador.

Se abrió PowerShell elevado y se repitió la operación.

### Lección aprendida

Antes de culpar al comando, servicio o sistema:

```text
¿la consola está elevada?
¿qué dice literalmente el error?
¿la operación requiere privilegios administrativos?
```

El contexto de ejecución forma parte del diagnóstico.

---

## 5. Instalación y validación de Sysmon

Con la característica habilitada se instaló Sysmon y se validó su funcionamiento.

Se comprobó el registro en:

```text
Visor de eventos
→ Registros de aplicaciones y servicios
→ Microsoft
→ Windows
→ Sysmon
→ Operational
```

Se observaron eventos Sysmon inmediatamente.

### Primeros eventos estudiados

```text
Event ID 1 → Process Create
Event ID 5 → Process Terminated
```

Ejemplos observados:

```text
powershell.exe
whoami.exe
backgroundTaskHost.exe
svchost.exe
```

---

## 6. Sysmon Event ID 1 — creación de proceso

Se generó actividad controlada con:

```powershell
cmd.exe /c whoami
powershell.exe -Command "Get-Date"
```

En Sysmon Event ID 1 se observaron campos como:

```text
Image:
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe

CommandLine:
"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -Command Get-Date

User:
BLUELAB\adm-mxnx

Hashes:
SHA256=...

ParentImage:
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

### Por qué sirve

Permite responder:

```text
¿qué proceso nació?
¿quién lo ejecutó?
¿con qué argumentos?
¿qué proceso padre lo lanzó?
¿cuál es su hash?
¿qué ProcessGuid identifica esa instancia concreta?
```

El hash funciona como una huella del ejecutable y el `ProcessGuid` permite seguir una instancia de proceso a través de distintos eventos.

---

## 7. Sysmon Event ID 5 — finalización de proceso

También se observaron eventos:

```text
Event ID 5 → Process Terminated
```

Ejemplos:

```text
Image: C:\Windows\System32\whoami.exe
User: BLUELAB\adm-mxnx
```

```text
Image: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
User: BLUELAB\adm-mxnx
```

### Concepto aprendido

Un proceso puede seguirse como una secuencia:

```text
Event 1
→ proceso creado

actividad intermedia
→ archivos / red / otros procesos según telemetría habilitada

Event 5
→ proceso finalizado
```

No se debe interpretar un Event ID aislado como malicioso. El valor está en el contexto y la correlación.

---

## 8. Revisión de configuración activa

Se consultó la configuración de Sysmon con:

```powershell
sysmon -c
```

Inicialmente no existía una configuración de filtrado propia y el monitoreo de red no estaba habilitado.

Se decidió crear una configuración mínima y comprensible antes de utilizar una configuración extensa de terceros.

---

## 9. Configuración XML mínima

Se creó:

```text
C:\Sysmon\sysmonconfig.xml
```

Contenido utilizado:

```xml
<Sysmon schemaversion="4.90">
  <HashAlgorithms>sha256</HashAlgorithms>

  <EventFiltering>
    <ProcessCreate onmatch="exclude">
    </ProcessCreate>

    <NetworkConnect onmatch="exclude">
    </NetworkConnect>

    <ProcessTerminate onmatch="exclude">
    </ProcessTerminate>
  </EventFiltering>
</Sysmon>
```

Se aplicó con:

```powershell
sysmon -c C:\Sysmon\sysmonconfig.xml
```

Después se verificó nuevamente:

```powershell
sysmon -c
```

Resultado relevante:

```text
Network connection: enabled

ProcessCreate      onmatch: exclude
NetworkConnect     onmatch: exclude
ProcessTerminate   onmatch: exclude
```

### Por qué partir con una configuración mínima

El objetivo de esta etapa es comprender la telemetría antes de optimizarla.

Una configuración grande puede esconder la lógica detrás de cientos de reglas. Primero se estudia:

```text
qué evento genera cada acción
qué campos entrega
qué ruido produce
qué filtros serán necesarios
```

Luego se endurece la configuración.

---

## 10. Sysmon Event ID 3 — conexiones de red

Con `NetworkConnect` habilitado comenzaron a generarse:

```text
Event ID 3 → Network connection detected
```

Campos observados:

```text
ProcessGuid
ProcessId
Image
User
Protocol
SourceIp
SourceHostname
SourcePort
DestinationIp
DestinationHostname
DestinationPort
DestinationPortName
```

### Ruido observado

Al registrar todas las conexiones aparecieron numerosos eventos legítimos, por ejemplo:

```text
msedgewebview2.exe → LAB-DC01:53
mmc.exe            → LAB-DC01:135
mmc.exe            → puertos RPC dinámicos
```

Esto demostró una realidad operativa:

> Más telemetría no significa automáticamente mejor detección. Sin filtros y contexto se genera ruido.

---

## 11. Primera búsqueda por atributos

Se realizó una prueba de conectividad SMB desde `LAB-CLIENT01` hacia el DC:

```powershell
Test-NetConnection 172.31.250.10 -Port 445
```

Resultado:

```text
SourceAddress    : 172.31.250.20
RemoteAddress    : 172.31.250.10
RemotePort       : 445
TcpTestSucceeded : True
```

En Sysmon se buscó específicamente:

```text
Event ID: 3
DestinationPort: 445
```

En vez de revisar manualmente cientos de eventos, se utilizó la búsqueda por atributos.

### Cambio de mentalidad

```text
Antes:
"buscar un Event ID 3"

Después:
"buscar un Event ID 3 con destino LAB-DC01 y puerto 445"
```

Esto introduce hunting básico basado en atributos.

---

## 12. Importancia de no atribuir sin evidencia suficiente

El primer evento encontrado hacia `445` mostraba:

```text
Image: System
ProcessId: 4
User: NT AUTHORITY\SYSTEM
SourceIp: 172.31.250.20
DestinationIp: 172.31.250.10
DestinationPort: 445
```

Aunque IP y puerto coincidían con la prueba, ese evento no demostraba por sí solo que PowerShell fuera el proceso responsable.

### Lección aprendida

No basta con que coincidan:

```text
hora
IP
puerto
```

También se debe revisar:

```text
Image
User
ProcessId
ProcessGuid
contexto de ejecución
```

En una investigación no se debe atribuir actividad a un proceso sin evidencia suficiente.

---

## 13. Prueba controlada para atribuir la conexión a PowerShell

Para generar una conexión claramente atribuible a la misma instancia de PowerShell se utilizó:

```powershell
$c = New-Object System.Net.Sockets.TcpClient
$c.Connect("172.31.250.10",445)
Start-Sleep 3
$c.Close()
```

Se obtuvo un Sysmon Event ID 3 con:

```text
Image:
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe

User:
BLUELAB\adm-mxnx

Protocol:
tcp

SourceIp:
172.31.250.20

SourceHostname:
LAB-CLIENT01.bluelab.test

DestinationIp:
172.31.250.10

DestinationHostname:
LAB-DC01

DestinationPort:
445

DestinationPortName:
microsoft-ds
```

Conclusión defendible:

> `BLUELAB\adm-mxnx`, mediante `powershell.exe` en `LAB-CLIENT01`, inició una conexión TCP hacia `LAB-DC01` por el puerto 445/SMB durante una prueba controlada del laboratorio.

---

## 14. Correlación mediante ProcessGuid

El Event ID 3 entregó un `ProcessGuid` asociado a la instancia de PowerShell.

Se intentó buscar un Event ID 1 con el mismo GUID.

Inicialmente no apareció.

### Causa encontrada

El PowerShell utilizado ya estaba abierto antes de que la telemetría necesaria quedara activa.

Por lo tanto podía existir el evento de red actual sin disponer del evento histórico de creación de ese mismo proceso dentro del período observado.

### Lección aprendida

> Para reconstruir una historia completa se necesita telemetría desde antes de que ocurra la actividad.

Activar monitoreo después puede entregar una parte de la historia, pero no necesariamente su comienzo.

---

## 15. Prueba limpia de correlación

Se cerró completamente el PowerShell anterior y se abrió uno nuevo con Sysmon ya operativo.

Primero se obtuvo el PID:

```powershell
$PID
```

Resultado observado en la prueba:

```text
7220
```

En esa misma instancia se generó la conexión:

```powershell
$c = New-Object System.Net.Sockets.TcpClient
$c.Connect("172.31.250.10",445)
Start-Sleep 3
$c.Close()
```

El Sysmon Event ID 3 mostró:

```text
ProcessId: 7220
Image: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
User: BLUELAB\adm-mxnx
SourceIp: 172.31.250.20
DestinationIp: 172.31.250.10
DestinationPort: 445
```

Además, los Event ID 1 generados al abrir la nueva consola mostraron procesos relacionados con esa instancia y referencias al mismo PID como proceso padre, por ejemplo `conhost.exe`:

```text
Image:
C:\Windows\System32\conhost.exe

ParentProcessId:
7220

ParentImage:
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe

ParentUser:
BLUELAB\adm-mxnx
```

Con esto se comprobó que Sysmon estaba observando la nueva instancia desde su nacimiento y podía relacionar su actividad posterior.

---

## 16. Qué se aprendió de correlación

La investigación dejó tres identificadores importantes:

```text
PID
→ identifica un proceso mientras existe, pero puede reutilizarse posteriormente

ProcessGuid
→ identifica de forma más estable una instancia de proceso dentro de Sysmon

ParentProcessGuid / ParentProcessId
→ permiten reconstruir relaciones padre-hijo
```

Flujo de análisis aprendido:

```text
Event ID 1
→ nace un proceso
→ revisar ProcessGuid / PID / CommandLine / Parent

Event ID 3
→ el proceso genera una conexión
→ revisar ProcessGuid / Image / IP / puerto

Event ID 5
→ el proceso termina
```

La correlación permite pasar de eventos aislados a una historia de comportamiento.

---

## 17. Comparación: auditoría nativa vs Sysmon

| Fuente | Evento | Qué aporta |
|---|---:|---|
| Windows Security | `4624` | inicio de sesión exitoso |
| Windows Security | `4625` | inicio de sesión fallido |
| Windows Security | `4672` | privilegios especiales |
| Windows Security | `4688` | creación de procesos y command line configurada |
| Sysmon | `1` | creación de proceso con GUID, hashes y contexto padre |
| Sysmon | `3` | conexión de red asociada a proceso |
| Sysmon | `5` | finalización de proceso |

No se trata de elegir una sola fuente. Las fuentes se complementan.

---

## 18. Método Blue Team consolidado

A esta altura del laboratorio el método repetible es:

```text
1. Definir qué quiero observar
2. Habilitar la telemetría necesaria
3. Verificar que realmente esté activa
4. Generar una acción controlada
5. Localizar la evidencia
6. Identificar usuario, proceso y host
7. Revisar línea de comandos
8. Revisar proceso padre
9. Revisar IP y puerto si hay red
10. Correlacionar por Logon ID, PID o ProcessGuid según corresponda
11. Separar ruido normal de actividad relevante
12. No atribuir sin evidencia suficiente
13. Documentar conclusión y límites de la evidencia
```

---

## 19. Habilidades demostradas en esta etapa

- instalación y validación de Sysmon en un endpoint Windows;
- navegación del canal `Microsoft-Windows-Sysmon/Operational`;
- creación y aplicación de una configuración XML mínima;
- habilitación de telemetría de red;
- interpretación de Sysmon Event IDs `1`, `3` y `5`;
- análisis de `Image`, `CommandLine`, `Hashes`, `User` y proceso padre;
- análisis de `SourceIp`, `DestinationIp` y `DestinationPort`;
- búsqueda manual de evidencia por atributos;
- diferenciación entre ruido legítimo y evidencia relacionada con una prueba;
- atribución de conexión de red a un proceso concreto;
- uso de PID y ProcessGuid como pivotes de investigación;
- comprensión de la importancia de disponer de telemetría previa al incidente.

---

## 20. Estado actual

El endpoint `LAB-CLIENT01` dispone ahora de dos capas de telemetría:

```text
AUDITORÍA NATIVA WINDOWS
4624 / 4625 / 4672 / 4688

            +

SYSMON
1 / 3 / 5
```

Esto permite observar:

```text
identidad
→ inicio de sesión
→ privilegios
→ ejecución de procesos
→ relación padre-hijo
→ línea de comandos
→ hashes
→ conexiones de red
→ finalización de procesos
```

---

## 21. Próximo paso

La configuración actual registra deliberadamente mucho tráfico para aprender.

El siguiente paso es **reducir ruido y diseñar una configuración Sysmon más profesional**, manteniendo evidencia útil y evitando enviar telemetría innecesaria al futuro SIEM.

Después:

```text
Sysmon afinado
        ↓
Wazuh / SIEM
        ↓
centralización de eventos
        ↓
reglas y alertas
        ↓
MITRE ATT&CK
        ↓
casos documentados de detección e investigación
```

---

## 22. Criterio profesional aprendido

La conclusión más importante de esta etapa no es memorizar `1`, `3` y `5`.

Es poder defender una investigación con evidencia:

```text
¿Qué pasó?
¿En qué equipo?
¿Con qué usuario?
¿Qué proceso estuvo involucrado?
¿Cómo nació?
¿Quién fue su padre?
¿Con qué argumentos?
¿A qué destino de red se conectó?
¿Por qué atribuyo esa actividad a ese proceso?
¿Qué parte de la historia puedo probar y cuál no?
```

Ese criterio es la base sobre la que luego trabajan un SIEM, un EDR y un analista SOC.

---

## 23. Primera reducción de ruido: excluir conexiones al puerto 53

Después de habilitar `NetworkConnect`, el volumen de eventos ID 3 aumentó rápidamente. Una fuente repetitiva eran conexiones hacia:

```text
DestinationPort: 53
```

El puerto `53` corresponde normalmente a DNS. En este laboratorio `LAB-CLIENT01` utiliza a `LAB-DC01` como servidor DNS, por lo que las consultas hacia el DC son frecuentes.

Para este ejercicio concreto se decidió reducir ese ruido en `NetworkConnect` sin deshabilitar la resolución DNS del sistema.

Se modificó `C:\Sysmon\sysmonconfig.xml`:

```xml
<NetworkConnect onmatch="exclude">
  <DestinationPort name="Ruido DNS hacia DC" condition="is">53</DestinationPort>
</NetworkConnect>
```

Se aplicó nuevamente:

```powershell
sysmon -c C:\Sysmon\sysmonconfig.xml
```

Y se verificó:

```powershell
sysmon -c
```

Resultado relevante:

```text
Network connection: enabled
NetworkConnect
  DestinationPort  filter: is  value: 53
```

### Validación de que no se perdió la evidencia útil

Se repitió la conexión controlada hacia SMB y continuó apareciendo un Event ID 3 con:

```text
Image: powershell.exe
User: BLUELAB\adm-mxnx
SourceIp: 172.31.250.20
DestinationIp: 172.31.250.10
DestinationPort: 445
DestinationPortName: microsoft-ds
```

Conclusión:

```text
se redujo ruido DNS del Event ID 3
sin eliminar la telemetría SMB usada en la investigación
```

### Precaución para producción

Excluir el puerto 53 de `NetworkConnect` no significa que DNS carezca de valor defensivo. Las consultas DNS pueden ser relevantes para detectar dominios sospechosos, C2 u otras actividades. La decisión tomada aquí pertenece al contexto de aprendizaje del laboratorio y deberá revisarse antes de adoptar una política de producción.

---

## 24. Fundamentos de red usados durante el análisis

Para interpretar correctamente Sysmon Event ID 3 se fijaron cuatro conceptos básicos.

### IP

Identifica un host o interfaz dentro de la red.

Ejemplo:

```text
172.31.250.20 → LAB-CLIENT01
172.31.250.10 → LAB-DC01
```

### Puerto

Identifica un punto lógico de comunicación asociado a un servicio o conexión.

Puertos encontrados o usados en el laboratorio:

```text
53   → DNS
135  → RPC Endpoint Mapper
445  → SMB / Microsoft-DS
```

Puertos que conviene empezar a reconocer:

```text
80    → HTTP
443   → HTTPS
3389  → RDP
```

### SourcePort y DestinationPort

En una conexión como:

```text
172.31.250.20:57000 → 172.31.250.10:445
```

se interpreta:

```text
172.31.250.20  → equipo origen
57000          → puerto local/efímero usado por el cliente
172.31.250.10  → equipo destino
445            → servicio destino SMB
```

El puerto origen suele ser temporal; el puerto destino suele ser más útil para entender qué servicio se intentó alcanzar.

### PID

`PID` significa `Process ID`.

Ejemplo:

```text
PID 9724 → una instancia concreta de powershell.exe
```

No debe confundirse con un puerto:

```text
PID     → identifica un proceso
Puerto  → identifica un punto de comunicación de red
IP      → identifica un host/interfaz
```

El PID puede reutilizarse posteriormente, por lo que Sysmon agrega `ProcessGuid` para mejorar la correlación entre eventos.

---

## 25. Evento de red con `<unknown process>`

Durante la revisión de Event ID 3 se observó también un registro con información incompleta, incluyendo:

```text
Image: <unknown process>
User: -
ProcessGuid: {00000000-0000-0000-0000-000000000000}
Protocol: tcp
SourceIp: 172.31.250.20
```

### Interpretación correcta

Esto no debe clasificarse automáticamente como malware.

La evidencia disponible permite afirmar solamente que Sysmon registró una conexión pero no pudo asociarla con información suficiente a un proceso concreto en ese evento.

El `ProcessGuid` compuesto por ceros refuerza que no existe una atribución válida de proceso disponible en ese registro.

Posibles explicaciones deben tratarse como hipótesis, no como hechos, por ejemplo:

```text
proceso de vida muy corta
contexto del proceso no disponible al registrar el evento
limitación o condición temporal de la telemetría
```

### Método de análisis ante un `<unknown process>`

No concluir por el nombre del campo. Revisar:

```text
1. fecha y hora
2. IP origen
3. IP destino
4. puerto origen
5. puerto destino
6. protocolo
7. eventos ID 1 / 3 / 5 cercanos
8. otros registros del endpoint
```

Conclusión metodológica:

> `unknown` significa atribución incompleta en ese evento, no actividad maliciosa confirmada.

---

## 26. Regla mental para leer Sysmon Event ID 3

Cada conexión de red debe leerse al menos en este orden:

```text
1. Image
   → ¿qué proceso?

2. User
   → ¿qué cuenta?

3. SourceIp
   → ¿desde qué host?

4. SourcePort
   → ¿qué puerto local/efímero?

5. DestinationIp
   → ¿hacia qué host?

6. DestinationPort
   → ¿hacia qué servicio?

7. ProcessId / ProcessGuid
   → ¿qué instancia concreta del proceso?
```

Ejemplo validado:

```text
BLUELAB\adm-mxnx
        ↓
powershell.exe
        ↓
LAB-CLIENT01 172.31.250.20:<puerto efímero>
        ↓ TCP
LAB-DC01 172.31.250.10:445
        ↓
SMB
```

Esta lectura transforma números aislados en una historia técnica entendible y defendible.

---

## 27. Estado después del primer afinamiento

La configuración mantiene:

```text
Event ID 1 → creación de procesos
Event ID 3 → conexiones de red
Event ID 5 → finalización de procesos
```

pero comienza a reducir ruido conocido:

```text
NetworkConnect
└── excluir DestinationPort 53
```

El criterio para los siguientes filtros será el mismo:

```text
observar primero
→ identificar ruido repetitivo
→ entender qué representa
→ decidir si conviene excluirlo
→ aplicar un cambio pequeño
→ validar que la evidencia útil siga apareciendo
```

No se filtrará telemetría a ciegas.