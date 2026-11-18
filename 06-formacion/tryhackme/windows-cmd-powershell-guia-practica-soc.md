# TryHackMe — Windows Command Line + Windows PowerShell: guía práctica para SOC N1

**Revisión:** 16 de noviembre de 2026  
**Origen:** ejercicios y capturas trabajados en TryHackMe, noviembre 2026.  
**Nivel:** fundamentos Windows, administración e investigación Blue Team.  
**Entornos:** máquinas efímeras de TryHackMe (nombres e IP cambian). Los ejemplos indicados como «propuestos» son ejercicios adicionales para practicar, no ejecuciones verificadas.

> **Objetivo de lectura:** cuando vuelva a este archivo, quiero recordar **qué estoy buscando, por qué ese comando sirve, cómo interpretar la salida y qué no puedo concluir**. No memorizar comandos aislados: encadenar **hipótesis → consulta → interpretación → comprobación → evidencia**.

## Índice
1. Distinguir CMD, PowerShell, SSH, proceso, servicio y PID
2. Sala Windows Command Line: conexión, sistema y red
3. Windows Command Line: archivos, procesos y apagado
4. Sala Windows PowerShell: objetos y descubrimiento de cmdlets
5. PowerShell: archivos, filtros y tuberías
6. PowerShell: procesos, servicios, conexiones y hashes
7. PowerShell Remoting / scripting: Invoke-Command
8. Ejercicios prácticos SOC con comandos listos para estudiar
9. Errores reales encontrados en la sala
10. Repaso acumulativo, preguntas y respuestas
11. Estado de avances / pendientes

---

## 1. Diccionario: qué es cada cosa

| Sigla / palabra | Significado en inglés | Explicación aplicada |
|---|---|---|
| CLI | Command-Line Interface | Interfaz de línea de comandos |
| GUI | Graphical User Interface | Interfaz gráfica |
| CMD | Command Prompt / cmd.exe | Intérprete tradicional de comandos Windows |
| PowerShell | Shell de automatización de Microsoft | Ejecuta **cmdlets** que devuelven **objetos** y permite scripts |
| Cmdlet | Command-let | Comando especializado, típicamente **Verbo-Sustantivo**, como Get-Service |
| SSH | Secure Shell | Protocolo para conectarse de forma cifrada a otro equipo |
| RDP | Remote Desktop Protocol | Protocolo de escritorio remoto gráfico |
| TCP | Transmission Control Protocol | Protocolo orientado a conexión |
| IP | Internet Protocol | Direcciones de red |
| DNS | Domain Name System | Traduce nombres a direcciones y ayuda a localizar servicios |
| DHCP | Dynamic Host Configuration Protocol | Asigna parámetros de red automáticamente |
| MAC | Media Access Control | Identificador de interfaz de red en capa de enlace |
| PID | Process Identifier | Número que identifica un proceso **en un momento concreto** |
| SMB | Server Message Block | Compartición de archivos y otros servicios Windows; TCP 445 habitual |
| RPC | Remote Procedure Call | Llamada a procedimiento remoto; endpoint mapper TCP 135 |
| RDP | Remote Desktop Protocol | Acceso remoto gráfico; TCP 3389 habitual |
| WinRM | Windows Remote Management | Administración remota; HTTP 5985 / HTTPS 5986 por defecto |
| SOC | Security Operations Center | Centro de operaciones de seguridad |
| IOC | Indicator of Compromise | Indicador que puede apoyar una hipótesis de compromiso |
| SHA-256 | Secure Hash Algorithm 256-bit | Huella de contenido que ayuda a comprobar integridad |
| ADS | Alternate Data Streams | Flujos alternativos de datos en NTFS, diferentes de Active Directory |
| NTFS | New Technology File System | Sistema de archivos Windows que permite atributos, ACL y ADS |
| SIEM | Security Information and Event Management | Centraliza y analiza eventos de seguridad |

**Distinciones básicas que se olvidan fácilmente:**

- **Proceso:** instancia de programa en ejecución, tiene PID. **Servicio:** componente administrado por el Service Control Manager; un servicio puede ejecutarse dentro de un proceso compartido.
- **Puerto:** punto lógico donde escucha o se conecta un programa. **PID:** qué proceso lo mantiene.
- **LISTENING:** acepta/intenta recibir conexiones, **no prueba que haya una sesión activa**. **ESTABLISHED:** sesión TCP establecida, **no prueba por sí misma que haya un ataque**.
- **CMD:** muchos comandos trabajan con salida textual. **PowerShell:** transporta objetos y propiedades en el pipeline (tubería).

## 2. Windows Command Line — acceso y reconocimiento

### 2.1 Iniciar VM + AttackBox

En la sala se utilizaron **Start Lab Machine** y **Start AttackBox**. Desde AttackBox se puede entrar por SSH en la máquina Windows. Nunca copiar la IP de capturas de ejemplo: **cada sesión puede recibir otra IP**.

Ejemplo genérico desde la terminal Linux del AttackBox:

~~~bash
ssh usuario@IP_ACTUAL_DE_LA_VM
~~~

- `ssh` establece la conexión segura.
- `usuario` es la cuenta indicada por la sala.
- `IP_ACTUAL_DE_LA_VM` se lee de «Active machines information».
- La primera conexión puede pedir confirmar huella de host: verificar que sea el destino del laboratorio antes de aceptarla.
- Escribir la contraseña temporal de TryHackMe cuando corresponda (no guardar contraseñas de laboratorio en GitHub).

Una vez dentro, el prompt puede verse como `user@WINSRV2022-CORE C:\Users\user>`. Significa que **estamos dentro de Windows remoto**, aunque la ventana está en Linux.

### 2.2 Recolectar información básica

~~~cmd
set
ver
systeminfo
~~~

| Comando | Búsqueda | Ejemplos observados |
|---|---|---|
| `set` | Variables de entorno, como `PATH`, `USERNAME`, `COMPUTERNAME` | `Path=C:\Windows\system32;...` |
| `ver` | Versión del kernel Windows | `10.0.20348.2655` en la VM anterior |
| `systeminfo` | Nombre, versión, instalación, memoria, tarjetas y parches | `WINSRV2022-CORE`, Windows Server 2022 Datacenter, WORKGROUP |
| `driverquery` | Drivers instalados | Listado de controladores |
| `help` o `comando /?` | Ayuda | Parámetros disponibles |
| `cls` | Limpiar consola | No borra registros del sistema |

**Qué vimos en esa VM:** `WINSRV2022-CORE`, `10.0.20348.2655`, alrededor de 2 GB de RAM, IP `10.67.152.15` y `WORKGROUP`. Son **datos de esa instancia**, no valores permanentes para otras máquinas.

**WORKGROUP:** el equipo no está unido a un dominio AD; Windows Server no implica automáticamente que sea un Domain Controller.

**PATH:** variable de entorno con carpetas que Windows busca cuando escribimos un ejecutable sin ruta absoluta. Separa carpetas con `;`. Una manipulación de PATH puede tener implicaciones de seguridad, pero no declarar compromiso sin evidencia.

### 2.3 Salida demasiado larga

~~~cmd
driverquery | more
~~~

- `|`: pipe, entrega salida al comando siguiente.
- `more`: muestra página por página; tecla Espacio avanza; Ctrl+C interrumpe.
- También se puede leer con `more archivo.txt`.

## 3. Windows Command Line — red

### 3.1 Información de la interfaz

~~~cmd
ipconfig
ipconfig /all
~~~

`ipconfig` muestra IP, máscara y gateway. `ipconfig /all` añade MAC (Physical Address), DNS, DHCP y detalles de adaptadores.

**Orden mental:** «¿Tengo dirección IP? ¿Gateway? ¿DNS? ¿DHCP?». No confundir falla de resolución DNS con falla de conectividad IP.

### 3.2 Conectividad, ruta y resolución DNS

~~~cmd
ping example.com
tracert example.com
nslookup example.com
nslookup example.com 1.1.1.1
~~~

| Herramienta | Qué consulta | Limitación |
|---|---|---|
| `ping` | Respuestas ICMP y tiempo aproximado | Un firewall puede bloquear ICMP aunque el servicio funcione |
| `tracert` | Saltos de red hacia destino | `*` puede significar ausencia de respuesta ICMP, no necesariamente corte de tráfico |
| `nslookup` | Resuelve nombre en DNS | DNS correcto no demuestra acceso al servicio |
| `nslookup dominio 1.1.1.1` | Pregunta a servidor DNS explícito | Útil para contrastar respuesta con DNS local |

### 3.3 Ver conexiones y puertos con netstat

~~~cmd
netstat
netstat -o
netstat -abno
~~~

| Opción | Significado |
|---|---|
| `-a` | Puertos en escucha y conexiones |
| `-b` | Ejecutable vinculado, requiere permisos suficientes |
| `-n` | No resuelve IP/puertos a nombres, todo numérico |
| `-o` | PID del proceso dueño de conexión |

Ejemplo interpretado en la práctica:

~~~text
TCP  10.67.152.15:22  10.67.79.254:40572  ESTABLISHED  1620
~~~

El servicio local usa puerto 22 (habitual SSH). Hay una sesión TCP establecida contra otra IP con puerto de origen efímero. `1620` era el PID **en esa sesión**, puede cambiar.

Buscar proceso por PID:

~~~cmd
tasklist /FI "PID eq 1620"
~~~

**Puertos vistos y aprendidos:**

| Puerto habitual | Función | Servicio Windows consultado en quiz |
|---|---|---|
| TCP 22 | SSH | sshd, según instalación |
| TCP 135 | RPC Endpoint Mapper | **RpcSs** (no contestar solamente «RPC» si pide nombre interno) |
| TCP 3389 | RDP / Escritorio remoto | **TermService** |
| TCP 445 | SMB | LanmanServer, según rol |
| TCP 5985 | WinRM HTTP | WinRM |
| TCP/UDP 88 | Kerberos | KDC en DC; no se asume presente en cualquier VM |

En la sala, `ipconfig /all` fue aceptado para la pregunta de MAC. Hubo corrección importante: el quiz de puertos pedía el **nombre interno del servicio**, distinto del protocolo. Consultar `sc query RpcSs` y `sc query TermService` en CMD ayuda a ver el servicio, si existe.

**Estados:** `LISTENING` = espera conexiones; `ESTABLISHED` = conexión TCP establecida; `SYN_SENT` = se ha enviado SYN y aún no se completa el handshake. No etiquetar «ataque» solo por un estado.

## 4. Windows Command Line — archivos, procesos y control del sistema

### 4.1 Navegar y leer

~~~cmd
cd \
dir
cd C:\Treasure\Hunt
dir
type flag.txt
~~~

**Ejercicio confirmado:** se llegó a `C:\Treasure\Hunt\flag.txt` y se leyó su contenido mediante `type`. Fue un aprendizaje nuevo: `type` permite mostrar texto sin Bloc de notas.

- `dir`: listar directorio.
- `cd`: cambiar ubicación.
- `type archivo.txt`: mostrar contenido del archivo.
- `nano` no es obligatorio ni viene instalado por defecto en ese Windows. Escribir un nombre `.txt` a secas intenta ejecutarlo/abrirlo, no es la forma portable de leerlo en CMD.
- `copy`, `move`, `del`/`erase`, `mkdir`, `rmdir`: administrar archivos/carpetas. Probar borrados solo en un directorio de pruebas.

### 4.2 Procesos: consultar antes de terminar

~~~cmd
tasklist
tasklist /FI "imagename eq notepad.exe"
taskkill /PID 1516
~~~

**Cómo leer el filtro:**
- `/FI` = Filter (filtro).
- `imagename` = nombre del ejecutable.
- `eq` = equals (igual a).
- `notepad.exe` = valor buscado.

En el ejercicio `tasklist /FI "imagename eq notepad.exe"` devolvió una instancia `notepad.exe`, PID 2548 (temporal).

**`taskkill /PID 1516` era respuesta de quiz**, no una recomendación para matar ese PID en producción. **No terminar procesos críticos ni usar `/F` sin identificar primero qué es y cuál será el impacto**.

### 4.3 Apagado, reinicio y mantenimiento

~~~cmd
shutdown /r
shutdown /a
~~~

- `/r` = restart: reinicia el equipo.
- `/a` = abort: cancela apagado/reinicio programado **si todavía hay ventana de espera**.
- `shutdown /s` = apagar.

**No ejecutar `shutdown /r` en servidores reales sin ventana autorizada.**

La conclusión de la sala también mencionó:

~~~cmd
chkdsk
driverquery
sfc /scannow
~~~

`chkdsk` examina sistema de archivos/disco; `sfc /scannow` verifica y puede reparar archivos del sistema. Los controles de reparación/corrección se hacen con autorización, respaldo y evaluación de impacto.

---

## 5. PowerShell — por qué es diferente a CMD

**PowerShell** combina intérprete de comandos, lenguaje de scripting y administración. Los **cmdlets** suelen usar formato **Verb-Noun (Verbo-Sustantivo)**:

| Cmdlet | Traducción |
|---|---|
| `Get-Process` | Obtener procesos |
| `Get-Service` | Obtener servicios |
| `Get-WinEvent` | Obtener eventos de Windows |
| `Get-ADUser` | Obtener usuario de Active Directory (cuando el módulo está disponible) |
| `New-Item` | Crear elemento |
| `Remove-Item` | Eliminar elemento |
| `Get-Content` | Leer contenido |
| `Invoke-Command` | Ejecutar comando, incluso de forma remota |

### 5.1 Objetos: propiedades y métodos

**Propiedad** = dato del objeto (`Name`, `Id`, `CPU`, `DisplayName`, `Length`).  
**Método** = operación del objeto (por ejemplo, `Kill()` sobre un objeto Process).

Ejemplo:

~~~powershell
Get-Process | Select-Object Name,Id,CPU
~~~

El resultado visible parece tabla, pero PowerShell **transportó objetos**, no texto plano. Por eso se puede filtrar por propiedades sin dividir cadenas manualmente.

### 5.2 Descubrir sintaxis, ayuda, alias

~~~powershell
Get-Command
Get-Command -Name Remove*
Get-Command -Verb Stop
Get-Help Get-Date
Get-Help New-LocalUser -Examples
Get-Alias
Get-Alias echo
~~~

- `Get-Command`: descubre comandos.
- `-Name Remove*`: nombres que empiezan con `Remove`; `*` es comodín (*wildcard*).
- `-Verb Stop`: cmdlets cuyo verbo es `Stop`, **no los ejecuta**.
- `Get-Help ... -Examples`: devuelve ejemplos de uso.
- `Get-Alias`: equivalencias abreviadas. `echo` es alias de `Write-Output`, `dir` de `Get-ChildItem`, `cd` de `Set-Location`, `type` y `cat` de `Get-Content` en PowerShell Windows.

**Error real del quiz:** `Get-Command -Verb Remove` es válido como consulta por verbo, pero la respuesta exacta pedida fue `Get-Command -Name Remove*`. `Get-Command -Verb Remove*` no era lo solicitado. **Aprendizaje:** distinguir un comando funcional de una respuesta exacta exigida por la plataforma.

### 5.3 Módulos

La sala introdujo `Find-Module` e `Install-Module` para buscar/instalar módulos (colecciones de cmdlets). **La máquina adjunta no tenía acceso a Internet para esas operaciones**; no asumir que se instalaron módulos en el laboratorio.

---

## 6. PowerShell — navegar, crear, leer, filtrar y ordenar

### 6.1 Navegación y archivos

~~~powershell
Get-ChildItem
Get-ChildItem -Path C:\Users
Set-Location -Path C:\Users
New-Item -Path .\PruebaSOC -ItemType Directory
New-Item -Path .\PruebaSOC\ejemplo.txt -ItemType File
Get-Content -Path .\PruebaSOC\ejemplo.txt
Copy-Item .\PruebaSOC\ejemplo.txt .\PruebaSOC\copia.txt
Move-Item .\PruebaSOC\copia.txt .\PruebaSOC\movido.txt
~~~

Estos comandos ilustran la sintaxis en un directorio de práctica, **no son afirmaciones de ejecución en tu VM**.

`Get-ChildItem` = enumerar, `Get-Content` = leer. Nunca confundir «encontrar un archivo» con «leer su contenido».

Para eliminar solo archivos de prueba, sabiendo exactamente qué ruta se modifica:

~~~powershell
Remove-Item -Path .\PruebaSOC\movido.txt
~~~

`-Path` indica la ruta. `.` = directorio actual; `..` = directorio padre. `-Force` en `Get-ChildItem -Force` incluye normalmente elementos ocultos y de sistema, **no otorga permisos de administrador**.

**Ejercicio realizado en TryHackMe:** listar `C:\Users`, leer `big-treasure.txt` en una carpeta oculta de `C:\Users\plr4t3`; `type` funcionó dentro de PowerShell por ser alias de `Get-Content`. Al contestar el número de elementos en `C:\Users`, había que ejecutar en la **VM real**, no contar las carpetas mostradas en la imagen de ejemplo de `C:\Users\captain`.

Contar elementos de un directorio:

~~~powershell
(Get-ChildItem -Path C:\Users | Measure-Object).Count
~~~

Por defecto no incluye elementos ocultos que requerirían `-Force`.

### 6.2 Pipeline (tubería) y comparadores

**`|`** = toma objetos de la izquierda y los envía a la derecha.

~~~powershell
Get-ChildItem | Sort-Object Length
Get-ChildItem | Select-Object Name,Length
Get-ChildItem | Where-Object -Property Length -gt 100
Get-ChildItem | Where-Object { $_.Extension -eq ".txt" }
~~~

En español, el tercero dice: «obtener elementos del directorio actual → quedarse con objetos cuyo tamaño sea mayor de 100 bytes». Los directorios pueden tener `Length` vacío/nulo; no confundir con tamaño calculado de una carpeta.

| Operador | Inglés | Español |
|---|---|---|
| `-eq` | equal | igual |
| `-ne` | not equal | distinto |
| `-gt` | greater than | mayor que |
| `-ge` | greater or equal | mayor o igual |
| `-lt` | less than | menor que |
| `-le` | less or equal | menor o igual |
| `-like` | pattern matching | coincide con patrón comodín |

Ejemplos concretos:

~~~powershell
# Archivos mayores a 100 bytes
Get-ChildItem | Where-Object -Property Length -gt 100

# Filtrar nombres que comienzan con ship
Get-ChildItem | Where-Object { $_.Name -like "ship*" }

# Ver archivos .log en carpetas anidadas
Get-ChildItem C:\Logs -Recurse -Filter *.log

# Leer solo últimas 20 líneas del archivo (si existe)
Get-Content C:\Logs\security.log -Tail 20

# Seguir nuevas líneas añadidas al archivo
Get-Content C:\Logs\security.log -Tail 20 -Wait
~~~

**`$_`** = «el objeto que está pasando ahora mismo por el filtro». `$_.Name` es su propiedad Name.

**Atención:** algunos ejemplos de `C:\Logs\security.log` son ejercicios conceptuales de repaso, no archivos probados en la VM.

### 6.3 Elegir mayor archivo de un directorio

~~~powershell
Get-ChildItem C:\Users\captain\Documents\captain-cabin -File |
  Sort-Object Length -Descending |
  Select-Object -First 1 Name,Length,FullName
~~~

- `-File` evita incluir carpetas.
- `Sort-Object Length -Descending`: mayor → menor.
- `Select-Object -First 1`: solo el primero.
- `FullName`: ruta absoluta.

Este fue un **desafío sugerido por la sala**, no se tiene captura de ejecución final.

### 6.4 Buscar texto dentro de un archivo

~~~powershell
Select-String -Path .\captain-hat.txt -Pattern "hat"
~~~

Busca texto; útil en logs y archivos de configuración. `Select-String` admite expresiones regulares (*regex*), pero un patrón simple basta para empezar.

**Diferencia:** `Get-ChildItem -Filter *.log` busca **nombres de archivos**; `Select-String` busca **contenido dentro de archivos**.

---

## 7. PowerShell — procesos, servicios, conexiones y huellas de archivo

### 7.1 Procesos

~~~powershell
Get-Process
Get-Process | Where-Object { $_.Name -eq "powershell" } | Select-Object Name,Id,CPU
~~~

`Get-Process` muestra procesos (no necesariamente qué servicio administra cada uno). `Id` = PID. `CPU` puede ser tiempo acumulado de procesador, **no necesariamente porcentaje instantáneo**.

### 7.2 Servicios: Name vs DisplayName, el reto del pirata

~~~powershell
Get-Service
Get-Service | Where-Object { $_.DisplayName -like "*merry*" } |
  Select-Object Name,DisplayName,Status
~~~

**Hallazgo realmente obtenido en la VM:**

~~~text
Name                DisplayName
plr4t3-s-compass    A merry life and a short one.
~~~

- **Name:** identificador interno `plr4t3-s-compass`.
- **DisplayName:** etiqueta amigable visible, modificada como pista del desafío.
- `-eq` requiere coincidencia exacta (se falló al omitir una «a» en la frase).
- `-like "*merry*"` permite coincidencia parcial sin recordar lema entero.

**Regla defensiva:** un DisplayName raro es **pista**, no prueba concluyente de persistencia maliciosa. Revisar binario, ruta, firma, cuenta de servicio, instalación/cambios y contexto.

### 7.3 Conexiones TCP y procesos propietarios

~~~powershell
Get-NetTCPConnection
Get-NetTCPConnection | Select-Object LocalAddress,LocalPort,RemoteAddress,RemotePort,State,OwningProcess
~~~

**OwningProcess** = propiedad que contiene el **PID** del proceso asociado.

Ejemplo de correlación si PID es 1620:

~~~powershell
Get-Process -Id 1620
~~~

**No confundir:** `Get-NetTCPConnection` devuelve conexiones; `OwningProcess` es el **nombre de la propiedad** que se preguntó en la sala. Es la idea equivalente a `netstat -o` de CMD.

### 7.4 Hashes: Get-FileHash

~~~powershell
Get-FileHash -Path .\big-treasure.txt
Get-FileHash -Path .\big-treasure.txt -Algorithm SHA256
~~~

En la práctica se obtuvo un hash `SHA256` de un archivo del desafío y TryHackMe lo aceptó. Un hash permite comparar **integridad de contenido**: si el hash cambia, probablemente cambiaron bytes del archivo (o procedimiento/entrada). Un hash por sí solo **no prueba que un archivo sea malware ni quién lo modificó**.

**Error observado:** se intentó consultar `ship-flag.txt` desde una carpeta distinta y `Resolve-Path` falló porque ese archivo no existía **en esa ruta actual**. Se cambió a `big-treasure.txt` de la ruta presente. La forma de investigar errores de ruta es `Get-Location`, `Get-ChildItem` y la ruta absoluta correcta.

### 7.5 ADS: Alternate Data Streams

La sala mostró un ejemplo *de otro equipo*:

~~~powershell
Get-Item -Path C:\House\house_log.txt -Stream *
~~~

Se veían `:$DATA` (flujo principal) y un flujo adicional `housinginfo`. **No existía ese ejemplo en la VM adjunta.**

**ADS** = *Alternate Data Streams* de NTFS. No confundir con **AD / Active Directory Domain Services**. Los ADS pueden usarse legítimamente o para ocultar contenido; encontrarlos requiere análisis de contexto.

---

## 8. Scripting y PowerShell Remoting: Task 8

**Scripting** = escribir instrucciones secuenciales en un archivo (por ejemplo `.ps1`) para ejecutarlas repetidamente y reducir tareas manuales.

En la sala se destacó su relevancia para:

- **Blue Team / SOC:** análisis de logs, IOC, detección de patrones y recopilación de evidencia.
- **Administración:** inventario, controles de integridad, salud de servicios y configuraciones.
- **Red Team autorizado:** pruebas de seguridad, simulaciones y evaluación de controles.

### 8.1 Invoke-Command: ejecutar instrucciones remotamente

La sintaxis del desafío fue:

~~~powershell
Invoke-Command -ComputerName RoyalFortune -ScriptBlock { Get-Service }
~~~

**Qué hace cada fragmento:**

| Fragmento | Interpretación |
|---|---|
| `Invoke-Command` | Ordena ejecutar instrucciones |
| `-ComputerName RoyalFortune` | El destino remoto se llama RoyalFortune |
| `-ScriptBlock { ... }` | Bloque de código a ejecutar en la máquina destino |
| `Get-Service` | Consulta servicios del equipo remoto |

**Sin `-Credential`** porque la pregunta decía asumir autenticación ya suficiente. Esto **no significa saltarse permisos**; normalmente se requiere PowerShell Remoting/WinRM habilitado, conectividad, identidad autorizada y configuración adecuada. En equipos fuera de dominio los requisitos pueden ser distintos.

### 8.2 Ejemplo Blue Team propuesto para .BLUELAB

~~~powershell
# EJEMPLO, NO VALIDADO AÚN EN .BLUELAB
Invoke-Command -ComputerName LAB-W11 -ScriptBlock {
    Get-NetTCPConnection |
      Select-Object LocalAddress,LocalPort,RemoteAddress,RemotePort,State,OwningProcess
}
~~~

Consulta las conexiones **dentro de LAB-W11**, y trae los resultados a nuestra consola. Esta fue una pregunta de repaso: la respuesta correcta era `Get-NetTCPConnection` dentro de las llaves.

**Lectura verbal:** «Invoco en LAB-W11 un bloque que obtiene conexiones TCP y devuelve IP, puertos, estado y PID». Antes de usarlo en producción, obtener autorización y validar que el acceso remoto es seguro.

---

## 9. Tres procedimientos SOC prácticos para repetir (propuestos)

### Caso A — ¿quién tiene una conexión sospechosa?

**Hipótesis:** una máquina mantiene una conexión no explicada hacia una IP externa.

1. Registrar hora y contexto.
2. Enumerar conexiones:

~~~powershell
Get-NetTCPConnection | Where-Object State -eq "Established" |
  Select-Object LocalAddress,LocalPort,RemoteAddress,RemotePort,OwningProcess
~~~

3. Tomar **OwningProcess** (PID) de la línea relevante:

~~~powershell
Get-Process -Id 1234 | Select-Object Name,Id,Path
~~~

4. Corroborar ruta de ejecutable (puede requerir permisos), firma, servicio, usuario y momento.
5. **Conclusión proporcionada:** conexión establecida y proceso observado. No declarar intrusión sin telemetría adicional.

**Evidencia a guardar:** salida consultada, fecha/hora, IP/puertos/PID y nombre/ruta; anonimizar datos de producción.

### Caso B — ¿existe un servicio disfrazado?

**Hipótesis:** un servicio tiene etiqueta visible inusual.

~~~powershell
Get-Service | Where-Object { $_.DisplayName -like "*merry*" } |
  Select-Object Name,DisplayName,Status
~~~

Después verificar configuración del servicio por nombre interno:

~~~powershell
Get-CimInstance Win32_Service -Filter "Name='plr4t3-s-compass'" |
  Select-Object Name,DisplayName,State,StartMode,PathName,StartName
~~~

**Segundo comando es ampliación propuesta, no ejecución ya documentada.** Si un servicio parece raro, revisar origen y binario. No eliminarlo inmediatamente.

### Caso C — ¿cambió un archivo que investigamos?

**Hipótesis:** un archivo del caso pudo haber cambiado.

~~~powershell
Get-FileHash -Algorithm SHA256 -Path C:\Ruta\archivo.txt
~~~

Guardar SHA-256 junto con nombre, ruta, tamaño, fecha y fuente. Comparar con un hash previo confiable **del mismo contenido**. Cambios no explican por sí solos autoría o intención.

---

## 10. Errores reales que conviene recordar

| Error | Causa | Qué revisar la próxima vez |
|---|---|---|
| Intentar `nano flag.txt` en CMD | Herramienta no instalada en ese Windows | `type` (CMD) o `Get-Content` (PowerShell) |
| `tasklist /FI notepad.exe` | Filtro incompleto | `tasklist /FI "imagename eq notepad.exe"` |
| Responder `RPC` / `RDP` a nombres de servicio | Protocolo ≠ nombre del servicio | `RpcSs` y `TermService` |
| `Get-Command -Verb Remove` no aceptado | Checker pedía forma con wildcard sobre nombre | `Get-Command -Name Remove*` |
| Contar carpetas de ejemplo `C:\Users\captain` para respuesta de `C:\Users` | Confusión entre imagen de referencia y salida de nuestra VM | Ejecutar y contar **en VM real** |
| `Get-FileHash` con ruta inexistente | Archivo de otra ruta o de otra VM | `Get-Location`, `Get-ChildItem`, ruta absoluta |
| `Get-Service ... -eq` no arrojó resultado | Frase escrita sin una `a` | `-like "*merry*"`, luego confirmar con texto exacto |
| Máquina TryHackMe reiniciada | VM efímera / tiempo límite | Releer IP actual, reconectar, no asumir persistencia |
| Contestar `tcpconnection` a propiedad | Confusión entre nombre del cmdlet y nombre de propiedad | `OwningProcess` = PID |

**Regla permanente:** las imágenes del lado izquierdo de TryHackMe pueden ser demostraciones; la ventana derecha suele ser la **ejecución real del usuario**. Antes de dar por hecho que un comando se ejecutó, mirar exactamente dónde aparece el prompt.

---

## 11. Autoevaluación acumulativa — 22 preguntas

Responder sin mirar el cuadro de soluciones; después explicar en voz alta **por qué**. Mezclar con preguntas antiguas de Active Directory, Kerberos y SOC.

1. ¿Qué hace `type` en CMD? ¿Cuál es su cmdlet equivalente?
2. ¿Qué hace `Get-ChildItem` y qué hace `Get-Content`?
3. ¿Qué indica `WORKGROUP` en `systeminfo`?
4. Expande GUI, CLI y SSH en inglés/español.
5. Diferencia `LISTENING` y `ESTABLISHED`.
6. Diferencia «puerto 3389» y «servicio TermService».
7. ¿Qué opción de netstat agrega PID? ¿Qué significa PID?
8. ¿Cómo obtendrías proceso del PID 1234 con CMD? ¿Con PowerShell?
9. ¿Qué propiedades aporta `Get-NetTCPConnection` para una investigación?
10. ¿Qué significa `OwningProcess`?
11. ¿Qué significa el pipe `|` en PowerShell?
12. ¿Qué es un objeto? ¿Qué es una propiedad? Da un ejemplo.
13. ¿Qué hacen `-eq`, `-gt`, `-lt`, `-like`?
14. ¿Qué significa `$_` dentro de un filtro?
15. ¿Qué haría `Get-ChildItem -Recurse -Filter *.log`?
16. ¿Qué haría `Get-Content security.log -Tail 20 -Wait`?
17. ¿Qué diferencia hay entre Name y DisplayName de un servicio?
18. ¿Qué nos permite demostrar un SHA-256 y qué no?
19. ¿Qué significa ADS aquí y qué relación tiene con NTFS?
20. ¿Dónde corre `Get-Service` en `Invoke-Command -ComputerName RoyalFortune -ScriptBlock { Get-Service }`?
21. Para un acceso SMB exitoso, ¿qué significa evento 4624 tipo 3?
22. Para investigar solicitudes TGS Kerberos, ¿qué evento del DC revisarías?

<details>
<summary>Respuestas breves — mirar solo después de intentar</summary>

1. Lee archivo; Get-Content.
2. Enumera elementos; lee contenido.
3. Equipo fuera de dominio AD.
4. Graphical User Interface, Command-Line Interface, Secure Shell.
5. Espera conexiones / conexión TCP establecida.
6. Puerto/protocolo habitual frente al nombre interno del servicio.
7. `-o`; Process Identifier.
8. `tasklist /FI "PID eq 1234"`; `Get-Process -Id 1234`.
9. IP/puertos locales y remotos, estado, propietario.
10. PID dueño de la conexión.
11. Envía objetos entre cmdlets.
12. Entidad de datos con propiedades y métodos; `Name`.
13. Igual, mayor, menor, coincide con patrón.
14. Objeto actual del pipeline.
15. Encuentra archivos .log en subcarpetas.
16. Lee últimas 20 líneas y sigue las nuevas.
17. Identificador interno / nombre visible.
18. Comparar integridad de bytes, no probar malignidad.
19. Alternate Data Streams en NTFS.
20. En equipo RoyalFortune si remoting está disponible/autorizado.
21. Inicio de sesión exitoso por red (no equivale por sí mismo a intrusión).
22. 4769 (ticket de servicio), correlacionar contexto.

</details>

### Rutina recomendada de repaso

- **Sesión de 5 minutos:** explicar 3 comandos sin mirarlos, luego comprobar con `Get-Help`.
- **Sesión de 15 minutos:** caso práctico A o B, guardando **salida real**, hipótesis y conclusión.
- **Al iniciar una nueva sala:** 2 preguntas antiguas + 1 nueva (repetición espaciada).
- **Para entrevista SOC N1:** contestar siempre «qué observé», «qué me falta validar» y «qué haría después».

---

## 12. Estado verificable y próximos pasos

**Aparece resuelto en capturas del usuario:**
- CMD: consulta versión y hostname, reconocimiento de sistema, lectura con `type`, ejercicio de puertos y consultas de procesos.
- PowerShell: reconocimiento de comandos y alias, lectura de archivo oculto del pirata, lectura `Get-FileHash` con SHA256 aceptado, búsqueda del servicio renombrado y respuesta correcta `plr4t3-s-compass`.
- PowerShell: preguntas teóricas de `Get-Content`, `Get-ChildItem -Path C:\Users`, `Write-Output`, `Get-Help ... -Examples` y conceptos de filtros.

**Contenido presentado, pero no demostrado como ejecutado por el usuario:**
- Consulta real PowerShell Remoting con `Invoke-Command` desde .BLUELAB.
- Caso SOC completo de correlación de red y proceso, análisis de ADS en su propia VM.
- Task 9 / Conclusion de la sala PowerShell: al 16/11/2026 se estaba continuando desde Task 8.

**Próximo desafío de portafolio propuesto:** desde `.BLUELAB`, inventariar procesos y servicios, conexiones TCP con `OwningProcess`, obtener hash SHA-256 de un archivo de prueba, y registrar resultados con capturas y una interpretación SOC («normal/sospechoso/indeterminado» con razones).

**Separar repositorios / tipos de evidencia:** TryHackMe = entrenamiento; .BLUELAB = práctica controlada propia; producción = nunca mezclar capturas, credenciales o datos sensibles.

---
**Idea para recordar:** `Get-ChildItem` encuentra; `Get-Content` lee; `Where-Object` filtra; `Sort-Object` ordena; `Select-Object` selecciona; `Get-Service` revisa servicios; `Get-NetTCPConnection` muestra conexiones; `OwningProcess` entrega PID; `Invoke-Command` ejecuta remotamente.
