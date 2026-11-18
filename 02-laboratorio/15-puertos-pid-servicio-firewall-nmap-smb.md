# Caso práctico SOC: puerto → PID → proceso → servicio → firewall → evidencia de DROP

Fecha: 30-10-2026  
Laboratorio: `BLUELAB / bluelab.test`

## Objetivo

Documentar paso a paso una investigación real del laboratorio donde se partió desde un puerto observado en Windows y se terminó correlacionando:

```text
puerto
→ PID
→ proceso
→ servicio
→ exposición de red
→ firewall
→ resultado Nmap
→ evidencia esperada de DROP
```

Este caso busca desarrollar una habilidad fundamental de SOC y troubleshooting:

> No quedarse con "el puerto está abierto" o "Nmap dice filtered", sino demostrar qué componente explica el comportamiento.

---

# 1. Método manual para encontrar qué hay detrás de un puerto

La lógica práctica que hay que recordar es:

```text
puerto → PID → proceso → servicio
```

No hace falta memorizar comandos complejos si se entiende esta cadena.

## 1.1 Ver conexiones y listeners

Opción clásica:

```cmd
netstat -ano
```

O con PowerShell:

```powershell
Get-NetTCPConnection -State Listen
```

### ¿Qué buscamos?

El puerto de interés y su `OwningProcess` / PID.

---

## 1.2 Convertir PID en proceso

Ejemplo investigado:

```text
PID 4516
```

Consulta:

```cmd
tasklist /FI "PID eq 4516"
```

Resultado observado:

```text
svchost.exe
```

### ¿Por qué esto todavía no basta?

Porque `svchost.exe` es un proceso contenedor genérico de Windows. Puede alojar distintos servicios.

---

## 1.3 Saber qué servicio vive dentro de svchost.exe

Comando:

```cmd
tasklist /svc /FI "PID eq 4516"
```

Interpretación de los parámetros:

```text
tasklist = lista procesos
/svc     = muestra servicios asociados
/FI      = aplica filtro
PID eq   = PID igual al valor indicado
```

Resultado del laboratorio:

```text
svchost.exe → PID 4516 → CDPSvc
```

Cadena completa:

```text
Puerto 5040
↓
PID 4516
↓
svchost.exe
↓
CDPSvc
```

---

# 2. Si no recuerdo el comando exacto

Tres búsquedas útiles en PowerShell:

```powershell
Get-Help *process*
Get-Command *service*
Get-Command *tcp*
```

Y en CMD siguen siendo muy útiles:

```cmd
netstat -ano
tasklist
tasklist /svc
```

## Lección

Para trabajo SOC vale más saber reconstruir:

```text
puerto → PID → proceso → servicio
```

que memorizar una lista enorme de cmdlets.

---

# 3. Caso 5040 — CDPSvc

Se comprobó localmente:

```text
Puerto: 5040
Estado: LISTEN
PID: 4516
Proceso: svchost.exe
Servicio: CDPSvc
```

CDPSvc corresponde a Connected Devices Platform Service.

## Conclusión correcta

No afirmar:

```text
"5040 es malo"
```

ni tampoco:

```text
"5040 es seguro porque pertenece a Windows"
```

La formulación correcta es:

> El puerto 5040 está asociado a un servicio legítimo de Windows, CDPSvc. Falta validar si ese listener concreto corresponde al comportamiento esperado de ese servicio en este host.

## Lección

```text
proceso legítimo != actividad automáticamente confiable
```

---

# 4. Validación local de 5040

En `LAB-CLIENT01`:

```powershell
Get-NetTCPConnection -LocalPort 5040 -State Listen |
Select-Object LocalAddress,LocalPort,OwningProcess
```

Luego:

```cmd
tasklist /svc /FI "PID eq 4516"
```

Y:

```powershell
Get-Service CDPSvc
```

Con esto se validan tres capas:

```text
puerto 5040
→ PID 4516
→ svchost.exe
→ CDPSvc
```

---

# 5. Validación desde Kali

Desde Kali:

```bash
nmap -n -Pn -p 5040 172.31.250.20
```

Resultado observado:

```text
5040/tcp filtered unknown
```

## ¿Qué significa filtered?

Nmap no recibió una respuesta suficiente para clasificar el puerto como `open` o `closed`.

Eso sugiere que algún control intermedio está filtrando o descartando las sondas.

La hipótesis principal en este caso fue:

```text
Windows Firewall
```

porque localmente el puerto estaba en LISTEN pero desde otra máquina no era accesible.

---

# 6. Error práctico detectado con Nmap

Se cometió primero este error:

```bash
nmap -n -Pn -p 172.31.250.20
```

El parámetro `-p` necesita un puerto.

Forma correcta:

```bash
nmap -n -Pn -p 5040 172.31.250.20
```

## Lección

```text
-p = puerto o lista de puertos
```

---

# 7. Segundo caso: SMB TCP/445

Se probó:

```bash
nmap -n -Pn -p 445 172.31.250.20
```

Resultado:

```text
445/tcp filtered microsoft-ds
```

## Importante

`microsoft-ds` es el nombre estándar asociado por Nmap al puerto 445.

No significa que Nmap haya conseguido establecer una sesión SMB.

Como el estado fue `filtered`, la comunicación no se completó.

---

# 8. Confirmación local de TCP/445

En Windows:

```powershell
Get-NetTCPConnection -LocalPort 445 -State Listen |
Select-Object LocalAddress,LocalPort,OwningProcess
```

Resultado observado:

```text
445 LISTENING
OwningProcess = 4
```

PID 4 corresponde a:

```text
System
```

## Interpretación

El servicio SMB está escuchando localmente y parte de su manejo está asociado al kernel/servicios del sistema.

---

# 9. Aparente contradicción

Desde Windows:

```text
TCP/445 = LISTEN
PID = 4
System
```

Desde Kali:

```text
445/tcp = filtered
```

Pregunta SOC:

> ¿Cómo puede un puerto estar escuchando localmente y aparecer filtrado desde otra máquina?

Respuesta:

Porque `LISTEN` solo demuestra que existe un listener local.

No demuestra que una conexión remota pueda atravesar:

- Windows Firewall;
- ACL;
- firewall de red;
- segmentación;
- otro control intermedio.

---

# 10. Revisar perfiles de Windows Firewall

Comando:

```powershell
Get-NetFirewallProfile |
Select-Object Name,Enabled,DefaultInboundAction,DefaultOutboundAction
```

En el laboratorio se observaron los tres perfiles habilitados.

## Qué queríamos saber

```text
¿Está el firewall activo?
¿qué comportamiento tiene para tráfico entrante?
```

Nota:

Que `DefaultInboundAction` aparezca como `NotConfigured` no significa automáticamente "permitir todo".

---

# 11. Buscar reglas SMB

Primera aproximación:

```powershell
Get-NetFirewallRule |
Where-Object DisplayName -like "*File and Printer Sharing*" |
Select-Object DisplayName,Enabled,Direction,Action
```

Alternativa clásica:

```cmd
netsh advfirewall firewall show rule name=all | findstr /I "File Printer SMB"
```

## Problema detectado

El sistema estaba en español.

Buscar texto en inglés dentro de `DisplayName` puede fallar por localización.

Por eso se decidió no depender del nombre visible de la regla.

---

# 12. Buscar directamente reglas habilitadas para LocalPort 445

Prueba usada:

```powershell
Get-NetFirewallRule -Enabled True -Direction Inbound -Action Allow |
Get-NetFirewallPortFilter |
Where-Object {$_.LocalPort -eq 445}
```

Resultado observado:

```text
sin salida
```

## Qué significa realmente

No se encontró una regla de entrada habilitada cuyo filtro de puerto fuera específicamente:

```text
LocalPort 445
```

## Lo que NO significa todavía

No demuestra por sí solo al 100 % que el firewall haya bloqueado la sonda concreta.

Una regla puede estar definida de otra manera:

- Any;
- servicio;
- aplicación;
- grupo;
- otra combinación de filtros.

Por eso hacía falta evidencia de logging.

---

# 13. Hipótesis de trabajo

Con la evidencia disponible:

```text
Windows:
445 LISTENING

Kali:
445 filtered

Firewall:
Enabled

Regla explícita Allow LocalPort 445:
no encontrada
```

Hipótesis más fuerte:

```text
Windows Firewall está filtrando la conexión entrante hacia TCP/445
```

Pero todavía faltaba la prueba definitiva.

---

# 14. Activar logging de paquetes bloqueados

En `LAB-CLIENT01`, PowerShell como administrador:

```powershell
Set-NetFirewallProfile -Profile Domain,Private,Public -LogBlocked True
```

Verificación:

```powershell
Get-NetFirewallProfile |
Select-Object Name,Enabled,LogBlocked,LogFileName
```

Ruta habitual:

```text
C:\Windows\System32\LogFiles\Firewall\pfirewall.log
```

---

# 15. Repetir la prueba desde Kali

Desde Kali:

```bash
nmap -n -Pn -p 445 172.31.250.20
```

Después, inmediatamente en Windows:

```powershell
Get-Content "C:\Windows\System32\LogFiles\Firewall\pfirewall.log" -Tail 30
```

## Evidencia que se buscaba

Una entrada equivalente conceptualmente a:

```text
DROP TCP 172.31.250.40 → 172.31.250.20:445
```

### Importante

Ese formato es el patrón esperado de evidencia.

No debe documentarse como resultado real hasta verificarlo efectivamente en el log.

---

# 16. Topología del caso

```text
KALI
172.31.250.40
      |
      | Nmap TCP/445
      v
LAB-CLIENT01
172.31.250.20
      |
      | localmente:
      | TCP/445 LISTEN
      | PID 4
      | System
      |
      v
Windows Firewall
      |
      └── filtrado esperado / DROP a comprobar
```

---

# 17. Método de investigación aprendido

La investigación no fue:

```text
"Nmap dice filtered, listo"
```

Fue:

```text
listener
→ PID
→ proceso
→ servicio
→ firewall
→ reglas
→ prueba remota
→ logging
→ evidencia
```

Ese método sirve para otros puertos y servicios.

---

# 18. Pregunta típica de entrevista SOC

## Pregunta

> Nmap muestra un puerto como filtered, pero en el endpoint aparece LISTENING. ¿Cómo lo explicas?

## Respuesta profesional

> LISTENING confirma que existe un socket local asociado a un proceso o servicio. FILTERED indica que desde el origen del escaneo no se obtuvo una respuesta que permita clasificarlo como open o closed. Revisaría controles intermedios, comenzando por firewall del host, reglas de entrada, ACL y segmentación, y buscaría evidencia en logs para demostrar qué componente está descartando el tráfico.

---

# 19. Diferencia entre LISTENING, open, closed y filtered

## LISTENING

Observación local del endpoint:

```text
hay un proceso esperando conexiones en ese puerto
```

## open

Desde el equipo remoto se logra determinar que el servicio acepta conexiones.

## closed

El host responde indicando que no existe un servicio escuchando en ese puerto.

## filtered

La sonda no obtiene suficiente respuesta; normalmente existe filtrado o descarte intermedio.

---

# 20. Qué quedó verificado en este ejercicio

### Puerto 5040

Verificado:

- listener local;
- PID 4516;
- proceso `svchost.exe`;
- servicio `CDPSvc`;
- desde Kali: `filtered`.

### Puerto 445

Verificado:

- listener local;
- PID 4;
- proceso/sistema: `System`;
- firewall habilitado;
- búsqueda específica de regla Allow/LocalPort 445 sin resultados;
- desde Kali: `filtered`.

### Pendiente en ese momento

Verificar directamente en:

```text
pfirewall.log
```

el paquete descartado correspondiente a la prueba desde Kali.

---

# 21. Cadena mental que debe quedar

Para cualquier puerto sospechoso o desconocido:

```text
1. ¿qué puerto?
2. ¿qué PID?
3. ¿qué proceso?
4. ¿qué servicio?
5. ¿qué usuario/contexto?
6. ¿está LISTENING?
7. ¿es accesible desde otra máquina?
8. ¿qué dice el firewall?
9. ¿qué dicen las reglas?
10. ¿qué dicen los logs?
11. ¿es esperado para ese host?
12. ¿qué evidencia respalda mi conclusión?
```

---

# 22. Valor para Blue Team / SOC N1

Este ejercicio entrena:

- enumeración local defensiva;
- interpretación de sockets;
- relación puerto/PID;
- relación proceso/servicio;
- análisis de `svchost.exe`;
- lectura de Windows Firewall;
- interpretación correcta de Nmap;
- validación cruzada endpoint/red;
- formulación de hipótesis;
- comprobación mediante logs;
- documentación basada en evidencia.

---

# 23. Resultado del caso

La habilidad importante no fue descubrir que "445 es SMB" o que "5040 pertenece a CDPSvc".

La habilidad real fue aprender a demostrar:

```text
qué existe localmente
+
qué ve una máquina remota
+
qué control explica la diferencia
```

Eso constituye una investigación técnica reproducible y defendible en un contexto SOC.

---

# 24. Próximo paso exacto del caso

Estado al cierre de esta bitácora:

```text
PENDIENTE DE VALIDACIÓN FINAL
```

Siguiente acción:

1. habilitar `LogBlocked`;
2. repetir Nmap TCP/445 desde Kali;
3. revisar `pfirewall.log`;
4. localizar la IP origen `172.31.250.40`;
5. confirmar `DROP TCP` hacia `172.31.250.20:445`;
6. guardar la evidencia;
7. cerrar la hipótesis como demostrada o descartada.

Solo después de esa evidencia debe escribirse:

```text
"Windows Firewall bloqueó la sonda"
```

como hecho comprobado.
