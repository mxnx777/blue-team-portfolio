# Wazuh SIEM: instalación, agente Windows, Sysmon y Threat Hunting

Fecha de laboratorio: 27-08-2026

## Objetivo

Levantar un servidor Wazuh en Ubuntu Server, integrar el endpoint Windows `LAB-CLIENT01`, validar conectividad, habilitar la recolección del canal de Sysmon y comprobar en Threat Hunting eventos `Sysmon Event ID 1` (creación de procesos).

---

## 1. Arquitectura del laboratorio

- Dominio: `bluelab.test`
- DC: `LAB-DC01`
- Cliente Windows: `LAB-CLIENT01`
  - IP: `172.31.250.20`
- Servidor Wazuh: `WAZUH-SRV01`
  - Ubuntu Server 24.04.4 LTS
  - IP interna: `172.31.250.30/24`
  - NIC interna: red VMware `LAB_ISOLADO`
  - NIC NAT adicional para acceso a Internet durante instalación
- Wazuh: instalación all-in-one
  - Wazuh manager
  - Wazuh indexer
  - Filebeat
  - Wazuh dashboard

Flujo esperado:

```text
LAB-CLIENT01
   ↓
Sysmon / Windows Event Log
   ↓
Wazuh Agent
   ↓
Wazuh Manager
   ↓
Indexer
   ↓
Threat Hunting / Dashboard
```

---

## 2. Instalación base de Ubuntu Server para Wazuh

Configuración de VM usada:

- 4 vCPU
- 6 GB RAM
- 60 GB disco
- Ubuntu Server 24.04.4 LTS
- OpenSSH habilitado
- LVM
- Hostname: `wazuh-srv01`

### Red interna

Configuración persistente vía Netplan para la NIC interna:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 172.31.250.30/24
      nameservers:
        addresses:
          - 172.31.250.10
        search:
          - bluelab.test
```

Para disponer de Internet durante la instalación se agregó una segunda NIC NAT, obteniendo una dirección `192.168.134.x/24` y una ruta por defecto vía VMware NAT.

Validaciones realizadas:

```bash
ip -br a
ip route
ping -c 3 1.1.1.1
ping -c 3 packages.wazuh.com
```

Resultado: conectividad IP y resolución DNS correctas.

---

## 3. Descarga e instalación de Wazuh

Error inicial detectado: se intentó usar `package.wazuh.com` en vez de `packages.wazuh.com`.

Descarga correcta:

```bash
curl -fL https://packages.wazuh.com/4.14/wazuh-install.sh -o /home/mono/wazuh-install.sh
```

Verificación:

```bash
ls -lh /home/mono/wazuh-install.sh
```

Instalación all-in-one:

```bash
sudo bash /home/mono/wazuh-install.sh -a
```

Al finalizar se verificó:

- Wazuh indexer instalado y activo
- Wazuh manager instalado y activo
- Filebeat instalado y activo
- Wazuh dashboard instalado y activo
- Dashboard disponible por HTTPS en puerto 443

Acceso desde el host:

```text
https://172.31.250.30
```

> Nota de seguridad: la contraseña generada para `admin` no debe registrarse en este repositorio.

---

## 4. Despliegue del agente en LAB-CLIENT01

Desde Wazuh Dashboard:

```text
Endpoints → Deploy new agent
```

Parámetros:

- Sistema: Windows
- MSI 32/64 bits
- Wazuh manager: `172.31.250.30`
- Agent name: `LAB-CLIENT01`
- Grupo inicial: `default`

Como el cliente no resolvía inicialmente `packages.wazuh.com`, el MSI se dejó localmente en:

```text
C:\wazuh-agent-4.14.7-1.msi
```

Instalación silenciosa:

```powershell
msiexec.exe /i C:\wazuh-agent-4.14.7-1.msi /q WAZUH_MANAGER="172.31.250.30" WAZUH_AGENT_NAME="LAB-CLIENT01"
```

Validación local:

```powershell
Get-Service WazuhSvc
Test-Path "C:\Program Files (x86)\ossec-agent"
```

Resultado esperado:

```text
WazuhSvc : Running
True
```

---

## 5. Validación de puertos entre cliente y manager

Desde `LAB-CLIENT01`:

```powershell
Test-NetConnection 172.31.250.30 -Port 1514
Test-NetConnection 172.31.250.30 -Port 1515
```

Resultado validado:

```text
TcpTestSucceeded : True
```

para ambos puertos.

Interpretación:

- TCP/1514: tráfico del agente hacia el manager.
- TCP/1515: enrolamiento/registro del agente.

El dashboard mostró posteriormente:

```text
LAB-CLIENT01
IP: 172.31.250.20
Status: active
Agent ID: 001
```

---

## 6. Validación inicial del agente

Desde la ficha del endpoint se comprobó:

- inventario de sistema
- Windows 11 Pro
- CPU/RAM
- Security Configuration Assessment
- benchmark CIS aplicado
- resultados de cumplimiento visibles

Ejemplo observado en SCA:

```text
Passed: 117
Failed: 356
Score aproximado: 24 %
```

Esto no equivale a “equipo comprometido”; representa cumplimiento contra controles de hardening del benchmark.

---

## 7. Comprobación local de Sysmon

En `LAB-CLIENT01`:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5 |
Select-Object TimeCreated, Id, Message
```

Se observaron, entre otros:

- Event ID 1 — Process Create
- Event ID 5 — Process Terminated

Se generó actividad controlada:

```powershell
whoami
ping 172.31.250.30
ipconfig
notepad.exe
```

---

## 8. Habilitar Sysmon en el agente Wazuh

Inicialmente Threat Hunting no devolvía eventos al filtrar por:

```text
data.win.system.channel
is
Microsoft-Windows-Sysmon/Operational
```

Conclusión: Sysmon estaba generando eventos localmente, pero el agente Wazuh no estaba recolectando ese canal.

Archivo editado:

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Antes de editar:

```powershell
Copy-Item "C:\Program Files (x86)\ossec-agent\ossec.conf" "C:\Program Files (x86)\ossec-agent\ossec.conf.bak"
```

Se agregó antes del cierre `</ossec_config>`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Reinicio:

```powershell
Restart-Service WazuhSvc
Get-Service WazuhSvc
```

---

## 9. Confirmación en Threat Hunting

Filtro 1:

```text
data.win.system.channel
is
Microsoft-Windows-Sysmon/Operational
```

Resultado observado: 22+ hits.

Filtro 2:

```text
data.win.system.eventID
is
1
```

Resultado observado: 23 eventos de creación de procesos.

Esto confirmó el circuito:

```text
proceso en Windows
→ Sysmon Event ID 1
→ Wazuh Agent
→ Wazuh Manager
→ regla/detección
→ Threat Hunting
```

---

## 10. Alertas observadas

En Threat Hunting aparecieron reglas/descripciones como:

- `Windows command prompt started by an abnormal process`
- `Discovery activity executed`
- `Powershell was used to delete files or directories`
- `SecEdit.exe binary in a suspicious location launched by ...powershell.exe`

También se observaron grupos como:

- `sysmon`
- `sysmon_eid1_detection`
- `windows`

Reglas visibles con niveles bajos/medios (por ejemplo 3–4), útiles para practicar triage.

### Lección clave

```text
ALERTA ≠ INCIDENTE
```

El analista debe revisar contexto:

1. usuario
2. proceso
3. línea de comandos
4. proceso padre
5. hora
6. host
7. relación con actividad esperada
8. si existe comportamiento repetitivo o encadenado

---

## 11. Campos a revisar en Sysmon Event ID 1

Al abrir un evento de Process Create en Wazuh conviene revisar:

```text
data.win.eventdata.image
data.win.eventdata.commandLine
data.win.eventdata.parentImage
data.win.eventdata.user
data.win.eventdata.processId
data.win.eventdata.parentProcessId
data.win.eventdata.hashes
data.win.system.eventID
data.win.system.channel
```

El objetivo es reconstruir:

```text
Quién
→ ejecutó qué
→ desde qué proceso padre
→ con qué argumentos
→ en qué endpoint
→ a qué hora
→ y por qué Wazuh generó la alerta
```

---

## 12. Errores útiles del laboratorio

### Error: URL mal escrita

```text
package.wazuh.com
```

Correcto:

```text
packages.wazuh.com
```

### Error: ruta incompleta al validar descarga

Incorrecto:

```bash
ls -lh /home/wazuh-install.sh
```

Correcto:

```bash
ls -lh /home/mono/wazuh-install.sh
```

### Error: PowerShell sin espacio en parámetro

Incorrecto:

```powershell
Test-NetConnection 172.31.250.30 -Port1514
```

Correcto:

```powershell
Test-NetConnection 172.31.250.30 -Port 1514
```

### Error visual en Wazuh: “No results match your criteria”

No significaba pérdida de datos. El rango temporal había quedado con inicio y fin prácticamente en el mismo instante.

Solución: volver a `Last 1 hour` o rango adecuado y reconstruir filtros.

---

## 13. Evidencia verificada en esta etapa

- Wazuh all-in-one instalado y dashboard accesible.
- `LAB-CLIENT01` registrado como agente activo.
- Comunicación TCP 1514 y 1515 validada.
- Sysmon Event Log activo localmente.
- Canal `Microsoft-Windows-Sysmon/Operational` agregado a `ossec.conf`.
- Eventos Sysmon visibles en Threat Hunting.
- Event ID 1 filtrado correctamente.
- Reglas de detección asociadas a procesos visibles en Wazuh.

---

## 14. Próximo paso

Realizar la primera investigación guiada de un evento:

1. seleccionar `Discovery activity executed`
2. revisar `Image`
3. revisar `CommandLine`
4. revisar `ParentImage`
5. revisar `User`
6. correlacionar con la acción controlada ejecutada
7. clasificar como:
   - benigno esperado
   - sospechoso
   - requiere escalamiento
8. documentar conclusión y evidencia

Después se avanzará a:
- reglas Wazuh personalizadas
- mapeo MITRE ATT&CK
- casos de detección reproducibles
- playbooks de triage
- incorporación del DC como segundo agente
- comparación entre Security Log y Sysmon


---

## 15. Primera investigación guiada — `Discovery activity executed`

Se abrió un evento desde Threat Hunting correspondiente a `Sysmon Event ID 1` y regla Wazuh:

```text
rule.description: Discovery activity executed
rule.id: 92031
rule.level: 3
rule.mitre.id: T1087
rule.mitre.tactic: Discovery
rule.mitre.technique: Account Discovery
```

### Evidencia verificada del evento

Endpoint:

```text
agent.name: LAB-CLIENT01
agent.ip: 172.31.250.20
data.win.system.computer: LAB-CLIENT01.bluelab.test
```

Proceso creado:

```text
data.win.eventdata.image:
C:\Windows\SysWOW64\net1.exe
```

Línea de comandos:

```text
C:\WINDOWS\system32\net1 user
```

Proceso padre:

```text
data.win.eventdata.parentImage:
C:\Windows\SysWOW64\net.exe
```

Línea de comandos del padre:

```text
net user
```

Usuario:

```text
NT AUTHORITY\SYSTEM
```

Directorio actual:

```text
C:\Program Files (x86)\ossec-agent\
```

Proceso:

```text
ProcessId: 1584
ParentProcessId: 4008
```

Canal y tipo:

```text
data.win.system.channel: Microsoft-Windows-Sysmon/Operational
data.win.system.eventID: 1
providerName: Microsoft-Windows-Sysmon
```

### Interpretación

`net user` enumera cuentas locales/del sistema, por lo que encaja con la técnica MITRE ATT&CK `T1087 — Account Discovery`.

La cadena observada fue:

```text
net.exe
  ↓
net1.exe
  ↓
net1 user
```

El evento fue ejecutado como `NT AUTHORITY\SYSTEM` y con directorio actual dentro de `C:\Program Files (x86)\ossec-agent\`.

### Clasificación de triage

**Verificado:** Wazuh detectó una acción de Account Discovery y la mapeó a MITRE T1087.

**Inferencia razonable:** por ejecutarse como `SYSTEM` y desde el directorio del agente Wazuh, este evento es probablemente actividad legítima generada por el propio agente o por una evaluación/configuración asociada al laboratorio, no una intrusión.

Clasificación provisional:

```text
Benigno esperado / actividad administrativa o de evaluación
```

No se debe cerrar solo por el nombre de la regla. En un entorno real se validaría además:

1. proceso padre completo;
2. hora exacta y tarea/programa que lo originó;
3. si existe actividad similar en otros endpoints;
4. si el comando fue ejecutado por una cuenta humana;
5. eventos correlacionados antes y después;
6. persistencia, movimiento lateral o escalamiento asociado.

### Lección SOC

Una regla de descubrimiento puede activarse por comportamiento legítimo.

```text
detección válida
≠
incidente confirmado
```

El trabajo del analista es contextualizar la telemetría y decidir si la actividad es esperada, sospechosa o maliciosa.


---

## 16. Segunda investigación guiada — PowerShell + File Deletion

Se abrió una alerta Sysmon Event ID 1 cuya regla Wazuh fue:

```text
rule.description: Powershell was used to delete files or directories
rule.id: 92021
rule.level: 3
rule.mitre.id: T1070.004
rule.mitre.tactic: Defense Evasion
rule.mitre.technique: File Deletion
```

### Evidencia verificada

Endpoint:

```text
agent.name: LAB-CLIENT01
agent.ip: 172.31.250.20
data.win.system.computer: LAB-CLIENT01.bluelab.test
```

Proceso:

```text
Image:
C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe

User:
NT AUTHORITY\SYSTEM

ProcessId:
8104
```

Proceso padre:

```text
ParentImage:
C:\Program Files (x86)\ossec-agent\wazuh-agent.exe

ParentCommandLine:
"C:\Program Files (x86)\ossec-agent\wazuh-agent.exe"

ParentProcessId:
5324

ParentUser:
NT AUTHORITY\SYSTEM
```

Directorio actual:

```text
C:\Program Files (x86)\ossec-agent\
```

Comando observado:

```powershell
secedit /export /cfg $env:TEMP\secpol.cfg;
Get-Content $env:TEMP\secpol.cfg | Select-String "ResetLockoutCount";
Remove-Item $env:TEMP\secpol.cfg
```

Canal:

```text
Microsoft-Windows-Sysmon/Operational
Event ID: 1
```

### Qué hizo realmente el comando

1. Exportó la política de seguridad local a un archivo temporal:
   `$env:TEMP\secpol.cfg`.
2. Leyó ese archivo y buscó el valor `ResetLockoutCount`.
3. Eliminó el archivo temporal con `Remove-Item`.

La alerta se activó porque PowerShell ejecutó una eliminación de archivo, comportamiento que Wazuh mapea a MITRE ATT&CK `T1070.004 - File Deletion`.

### Triage

**Verificado:** hubo una eliminación de archivo temporal mediante PowerShell.

**Contexto fuerte de legitimidad:** el proceso padre fue directamente `wazuh-agent.exe`, se ejecutó como `NT AUTHORITY\SYSTEM` y el comando parece orientado a consultar una configuración de política local y luego limpiar el archivo temporal generado.

Clasificación provisional:

```text
Benigno esperado / actividad de evaluación del agente Wazuh
```

### Lección SOC

La misma técnica ATT&CK puede aparecer tanto en actividad maliciosa como en tareas legítimas.

```text
PowerShell + Remove-Item
→ comportamiento observable válido
→ regla válida
→ contexto necesario
→ no implica compromiso por sí solo
```

El valor analítico está en reconstruir la cadena padre-hijo y entender la finalidad del comando.


---

## 17. Correlación temporal — View surrounding documents

Se utilizó **View surrounding documents** sobre la alerta de PowerShell/File Deletion para observar eventos inmediatamente anteriores y posteriores.

### Secuencia observada

En la ventana temporal alrededor de las 15:11:53 se observó una cadena coherente con una comprobación de política local iniciada por Wazuh:

```text
wazuh-agent.exe
  ↓
powershell.exe
  ↓
SecEdit.exe /export /cfg C:\Windows\TEMP\secpol.cfg
  ↓
conhost.exe
  ↓
PowerShell lee secpol.cfg y busca ResetLockoutCount
  ↓
Remove-Item secpol.cfg
```

Campos visibles en los eventos correlacionados mostraron:

- `agent.name: LAB-CLIENT01`
- `agent.ip: 172.31.250.20`
- `manager.name: wazuh-srv01`
- `parentProcessName` asociado a PowerShell/Wazuh
- `newProcessName` para `SecEdit.exe`, `conhost.exe` y PowerShell
- `commandLine` con la exportación de política y lectura de `ResetLockoutCount`
- Event ID de creación de procesos y eventos relacionados en el mismo intervalo temporal

### Interpretación

La vista de contexto temporal permitió comprobar que la alerta de eliminación de archivo no fue un evento aislado. Formó parte de una secuencia de evaluación:

1. Wazuh inicia PowerShell.
2. PowerShell ejecuta `secedit` para exportar la política.
3. Se crea la consola auxiliar.
4. Se consulta el parámetro de bloqueo.
5. Se elimina el archivo temporal.

Esto refuerza la clasificación provisional como **actividad legítima del agente / evaluación de configuración**, no como borrado malicioso.

### Lección SOC

```text
evento aislado
≠
historia completa
```

La correlación temporal ayuda a reconstruir intención y causalidad. Antes de escalar una alerta, revisar los eventos cercanos puede distinguir entre:

- herramienta de seguridad;
- script administrativo;
- actividad de usuario;
- ejecución realmente sospechosa.


---

## 18. Comparación de telemetría — Windows Security 4688 vs Sysmon Event ID 1

Durante un ejercicio de Discovery con comandos como `hostname` e `ipconfig`, se observó que Wazuh también estaba recibiendo eventos desde el registro nativo de seguridad de Windows.

Evento verificado:

```text
data.win.system.channel: Security
data.win.system.eventID: 4688
providerName: Microsoft-Windows-Security-Auditing
rule.description: A process was created.
```

### Significado

`Security Event ID 4688` indica que Windows registró la creación de un nuevo proceso mediante su auditoría nativa.

Esto debe distinguirse de:

```text
Sysmon Event ID 1
= Process Create registrado por Sysmon
```

Ambas fuentes pueden describir la creación de procesos, pero provienen de telemetría distinta:

```text
Windows Security Log
  └─ Event ID 4688
     └─ auditoría nativa de Windows

Sysmon Operational
  └─ Event ID 1
     └─ telemetría enriquecida de Sysmon
```

### Lección SOC

Un mismo comportamiento puede observarse desde varias fuentes. El analista debe identificar:

1. canal/origen del evento;
2. Event ID;
3. proveedor;
4. campos disponibles;
5. contexto de proceso;
6. correlación entre fuentes.

La existencia de múltiples eventos no implica múltiples ataques: puede tratarse del mismo proceso registrado por distintas fuentes de telemetría.


### Evidencia práctica del mismo proceso en Sysmon

Se localizó en el Visor de eventos el mismo proceso `ipconfig.exe` bajo:

```text
Microsoft-Windows-Sysmon/Operational
Event ID: 1
```

Campos visibles en el evento Sysmon:

```text
Image: C:\Windows\System32\ipconfig.exe
CommandLine: "C:\WINDOWS\system32\ipconfig.exe"
User: BLUELAB\adm-mxnx
IntegrityLevel: High
OriginalFileName: ipconfig.exe
Hashes: SHA256=...
ProcessGuid: {...}
ParentProcessGuid: {...}
```

Comparación observada:

```text
Windows Security 4688
- NewProcessName
- CommandLine
- ParentProcessName
- SubjectUserName
- MandatoryLabel

Sysmon Event ID 1
- Image
- CommandLine
- User
- IntegrityLevel
- Hashes
- ProcessGuid
- ParentProcessGuid
- metadatos del binario (Description, Product, Company, OriginalFileName)
```

Conclusión: ambos eventos registran creación de proceso, pero Sysmon entrega telemetría enriquecida especialmente útil para correlación, identificación del binario y reconstrucción del árbol de procesos.


---

## 19. Activación de Wazuh Archives y validación de telemetría cruda

Se habilitó la conservación e indexación de eventos que no necesariamente generan alertas.

Configuración aplicada en el Wazuh Manager:

```xml
<logall_json>yes</logall_json>
```

Configuración aplicada en Filebeat:

```yaml
alerts:
  enabled: true

archives:
  enabled: true
```

Se reiniciaron y validaron:

```text
wazuh-manager: active
filebeat: active
filebeat test output: OK
TLS handshake: OK
talk to server: OK
```

### Validación con ipconfig.exe

Desde `LAB-CLIENT01` se ejecutó nuevamente:

```powershell
ipconfig
```

Luego, en `WAZUH-SRV01`, se comprobó:

```bash
sudo grep -i "ipconfig.exe" /var/ossec/logs/archives/archives.json | tail -5
```

Resultado: el evento apareció en `archives.json`.

La evidencia contenía, entre otros:

```text
agent.name: LAB-CLIENT01
channel: Microsoft-Windows-Sysmon/Operational
eventID: 1
Image: C:\Windows\System32\ipconfig.exe
CommandLine: "C:\WINDOWS\system32\ipconfig.exe"
User: BLUELAB\adm-mxnx
IntegrityLevel: High
ParentImage: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

También se observaron eventos equivalentes provenientes de Windows Security, demostrando que la misma ejecución puede existir en múltiples fuentes de telemetría.

### Conclusión

El evento `ipconfig.exe` existe aunque no haya generado una alerta visible en `wazuh-alerts-*`.

Flujo validado:

```text
LAB-CLIENT01
  ↓
Sysmon Event ID 1
  ↓
Wazuh Agent
  ↓
Wazuh Manager
  ↓
archives.json
  ↓
Filebeat
  ↓
Wazuh Indexer
  ↓
wazuh-archives-*
```

Esto permite threat hunting sobre telemetría cruda y no solamente sobre alertas.


---

## 19. Wazuh Archives — telemetría cruda validada e indexada

Se habilitó la conservación e indexación de eventos que no necesariamente generan alertas.

Configuración aplicada:

```text
/var/ossec/etc/ossec.conf
<logall_json>yes</logall_json>

/etc/filebeat/filebeat.yml
archives:
  enabled: true
```

Después de reiniciar `wazuh-manager` y `filebeat`, se validó Filebeat con:

```bash
sudo filebeat test output
```

Resultado: conexión OK, TLS OK y comunicación con el Wazuh Indexer correcta.

### Prueba con ipconfig.exe

Desde `LAB-CLIENT01` se ejecutó nuevamente:

```powershell
ipconfig
```

En `WAZUH-SRV01` se confirmó el evento en:

```text
/var/ossec/logs/archives/archives.json
```

La telemetría contenía:

```text
agent.name: LAB-CLIENT01
channel: Microsoft-Windows-Sysmon/Operational
eventID: 1
image: C:\Windows\System32\ipconfig.exe
user: BLUELAB\adm-mxnx
parentImage: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
integrityLevel: High
```

### Confirmación en el Indexer

Consulta:

```bash
curl -k -u admin 'https://127.0.0.1:9200/_cat/indices/wazuh-archives-*?v'
```

Se observaron índices `green/open`, incluyendo:

```text
wazuh-archives-4.x-2026.09.22
wazuh-archives-4.x-2026.09.23
```

Con documentos indexados y tamaño creciente, confirmando el flujo completo:

```text
LAB-CLIENT01
→ Sysmon
→ Wazuh Agent
→ Wazuh Manager
→ archives.json
→ Filebeat
→ Wazuh Indexer
→ wazuh-archives-*
```

### Lección SOC

```text
EVENTO != ALERTA != INCIDENTE
```

`wazuh-alerts-*` contiene eventos que dispararon reglas.
`wazuh-archives-*` permite consultar telemetría recibida aunque no haya generado alerta.

Próximo paso: crear el index pattern `wazuh-archives-*` en Wazuh Dashboard y buscar `ipconfig.exe` desde Discover.


### Correlación completa del ciclo de vida de ipconfig.exe

Se confirmó en Discover sobre `wazuh-archives-*` la correlación entre creación y término de la misma instancia de `ipconfig.exe`.

Para la instancia ejecutada aproximadamente a las 13:47 hora local:

```text
Sysmon Event ID 1 = Process Creation
ProcessGuid = {67c6ec4f-029d-6ab4-6106-000000001400}
ProcessId = 9740
ParentImage = powershell.exe
User = BLUELAB\adm-mxnx
```

Luego se consultó:

```text
channel = Microsoft-Windows-Sysmon/Operational
eventID = 5
processGuid = {67c6ec4f-029d-6ab4-6106-000000001400}
```

Resultado: 1 hit correspondiente a la terminación del mismo proceso.

La búsqueda exacta requirió conservar las llaves `{ }` del valor `ProcessGuid`.

Conclusión:

```text
Event ID 1 + mismo ProcessGuid = nacimiento del proceso
Event ID 5 + mismo ProcessGuid = término del mismo proceso
```

Esto demuestra correlación por GUID y ciclo de vida de proceso usando telemetría Sysmon almacenada en Wazuh Archives.


---

## 20. Caso 2 — Discovery con PowerShell, net.exe/net1.exe y whoami /groups

Se ejecutaron de forma controlada en `LAB-CLIENT01`:

```powershell
net user
whoami /groups
```

La investigación comenzó en el Visor de eventos, canal `Microsoft-Windows-Sysmon/Operational`, filtrando `Event ID 1` (Process Creation), y luego se cotejó en Wazuh Archives/Discover.

### Cadena observada

```text
powershell.exe
  ├─> net.exe
  │     └─> net1.exe
  └─> whoami.exe /groups
```

La cuenta observada fue:

```text
BLUELAB\adm-mxnx
```

Los eventos mostraron `IntegrityLevel: High`, indicando ejecución con token elevado.

### Interpretación

- `net user`: enumeración de cuentas locales.
  - MITRE ATT&CK: `T1087.001 - Account Discovery: Local Account`
  - Táctica: `Discovery`
- `whoami /groups`: consulta de pertenencia a grupos del usuario actual.
  - MITRE ATT&CK principal para grupos locales: `T1069.001 - Permission Groups Discovery: Local Groups`
  - El uso de `whoami` también se relaciona con `T1033 - System Owner/User Discovery` cuando el objetivo es identificar al usuario actual.

### Correlación

Se usaron `ProcessGuid` y eventos Sysmon para confirmar creación y término de procesos. En particular, se observó el mismo `ProcessGuid` en eventos de creación y terminación para `net1.exe` y `whoami.exe`.

Lección principal: no asumir que `net1.exe` nació directamente de PowerShell. La cadena observada fue `powershell.exe -> net.exe -> net1.exe`, mientras que `whoami.exe /groups` fue lanzado directamente desde PowerShell.

Esto refuerza la lectura de:
- `Image`
- `CommandLine`
- `ParentImage`
- `ParentCommandLine`
- `User`
- `IntegrityLevel`
- `ProcessId`
- `ParentProcessId`
- `ProcessGuid`
- `ParentProcessGuid`


---

## 21. Caso 3 — PowerShell con EncodedCommand (Base64) y triage

Se generó de forma controlada en `LAB-CLIENT01` un PowerShell hijo con comando codificado en Base64:

```powershell
$cmd = 'Write-Output "BLUELAB-TEST"'
$enc = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
powershell.exe -NoProfile -EncodedCommand $enc
```

Sysmon registró la creación del proceso y Wazuh generó una alerta asociada a PowerShell con comando codificado.

Datos observados:
- Usuario: `BLUELAB\adm-mxnx`
- Integridad: `High`
- Proceso: `powershell.exe`
- Proceso padre: `powershell.exe`
- Canal: `Microsoft-Windows-Sysmon/Operational`
- Sysmon Event ID: `1`
- MITRE ATT&CK: `T1059.001 - PowerShell`
- Táctica: `Execution`
- Regla Wazuh observada: `92057`
- Nivel: `12`

La cadena Base64 fue extraída desde `CommandLine` y decodificada de forma segura con:

```powershell
[Text.Encoding]::Unicode.GetString([Convert]::FromBase64String($enc))
```

Resultado:

```text
Write-Output "BLUELAB-TEST"
```

Conclusión del triage:
- La detección fue correcta: el comportamiento observado coincidió con una condición sospechosa real (PowerShell + EncodedCommand).
- En este laboratorio, el contenido decodificado fue benigno y autorizado.
- La alerta es un verdadero positivo de la regla de detección, pero no constituye por sí sola un incidente malicioso.
- El contexto (usuario, horario, endpoint, proceso padre, contenido del comando y actividad posterior) es necesario antes de escalar a incidente.
