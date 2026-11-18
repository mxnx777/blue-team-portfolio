# COMANDOS — Levantamiento y verificación técnica (Windows Server)

Propósito:
Registrar comandos usados durante levantamientos de ciberseguridad,
explicar qué hacen, cómo interpretar sus resultados
y permitir repetir el análisis con criterio técnico.
Este archivo NO es bitácora (la bitácora va aparte).

================================================================
ORDEN DE EJECUCIÓN RECOMENDADO
================================================================
1) Identidad / Dominio
2) Sistema y Hardware
3) Red
4) Puertos y servicios
5) HTTP / IIS
6) Acceso remoto
7) Usuarios y privilegios
8) SMB
9) Validación de hosts (ARP)

================================================================
1) IDENTIDAD DEL SERVIDOR (AD / STANDALONE)
================================================================

Comando:
wmic computersystem get domain,partofdomain,name

Para qué sirve:
Determinar si el servidor pertenece a un dominio Active Directory
o funciona como servidor standalone (workgroup).

Interpretación:
- PartOfDomain = TRUE  
  → El servidor pertenece a un dominio.
- PartOfDomain = FALSE  
  → El servidor es standalone (NO AD).

------------------------------------------------

Comando:
wmic computersystem get domain,domainrole,name

Para qué sirve:
Determinar si el servidor cumple rol de Domain Controller.

Interpretación de DomainRole:
- 4 → Backup Domain Controller (BDC)
- 5 → Primary Domain Controller (PDC)
- 0 / 1 / 2 / 3 → NO es Domain Controller

Conclusión típica:
Si NO es DC, el impacto de un incidente es menor
que en un controlador de dominio.

================================================================
2) SISTEMA Y HARDWARE (CAPACIDAD REAL DEL SERVIDOR)
================================================================

Comando:
systeminfo | findstr /B /C:"Nombre del sistema operativo"

Para qué sirve:
Identificar versión exacta del sistema operativo.

------------------------------------------------

Comando:
systeminfo

Para qué sirve:
Obtener información completa del sistema:
SO, parches, arquitectura, memoria y hardware base.

------------------------------------------------

Comando:
wmic cpu get name,NumberOfCores,NumberOfLogicalProcessors

Para qué sirve:
Identificar modelo de CPU y capacidad de procesamiento.

Interpretación:
Más cores / threads = mayor capacidad de carga y recuperación.

------------------------------------------------

Comando:
wmic computersystem get TotalPhysicalMemory

Para qué sirve:
Obtener memoria RAM total instalada.

Interpretación:
RAM baja = riesgo en restauraciones, servicios y contingencia.

------------------------------------------------

Comando:
wmic logicaldisk get name,freespace,size,filesystem

Para qué sirve:
Listar discos, espacio libre, tamaño total y sistema de archivos.

Interpretación:
- Poco espacio libre → riesgo operativo
- NTFS esperado en servidores Windows

================================================================
3) CONFIGURACIÓN DE RED
================================================================

Comando:
ipconfig /all

Para qué sirve:
Ver configuración de red completa:
IP fija o DHCP, gateway, DNS, NetBIOS e interfaces activas.

Interpretación:
- IP fija → correcto para servidores
- DNS externo → anotar dependencia
- NetBIOS activo → legado, evaluar más adelante

================================================================
4) PUERTOS Y SERVICIOS ESCUCHANDO
================================================================

Comando:
netstat -ano | findstr LISTEN

Para qué sirve:
Ver todos los puertos que el servidor está escuchando.

Interpretación:
Un puerto abierto NO es un problema por sí solo;
importa el servicio que lo usa y si está expuesto.

------------------------------------------------

Comando:
netstat -ano | findstr ":80"
tasklist /FI "PID eq 4"

Para qué sirve:
Identificar el proceso dueño del puerto 80.

Interpretación:
- PID 4 → System (kernel / HTTP.sys)
- Normal en Windows Server
- NO indica malware

================================================================
5) HTTP / IIS
================================================================

Comando:
netsh http show servicestate

Para qué sirve:
Ver qué servicios están registrados en HTTP.sys.

Interpretación:
Permite confirmar si el puerto 80 pertenece a:
- IIS
- WinRM
- Otro servicio legítimo

------------------------------------------------

Comando:
sc query W3SVC

Para qué sirve:
Ver estado del servicio IIS (World Wide Web Service).

Interpretación:
- RUNNING → IIS activo
- STOPPED → IIS instalado pero detenido

================================================================
6) ACCESO REMOTO
================================================================

Comando:
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections

Para qué sirve:
Ver si RDP está habilitado o deshabilitado.

Interpretación:
- 1 → RDP DESHABILITADO (correcto si no se usa)
- 0 → RDP HABILITADO (evaluar riesgo)

------------------------------------------------

Comando:
winrm enumerate winrm/config/listener

Para qué sirve:
Ver listeners activos de WinRM.

Interpretación:
- Escucha solo en LAN → bajo riesgo
- Escucha en 0.0.0.0 → revisar firewall

================================================================
7) USUARIOS Y PRIVILEGIOS LOCALES
================================================================

Comando:
net user

Para qué sirve:
Listar todos los usuarios locales del servidor.

Interpretación:
- Usuarios obsoletos → riesgo
- Usuarios genéricos → riesgo

------------------------------------------------

Comando:
net localgroup administrators

Para qué sirve:
Listar quién tiene privilegios administrativos totales.

Interpretación:
- Pocos admins → correcto
- Muchos admins → riesgo interno

================================================================
8) SMB (PROTOCOLOS DE ARCHIVOS)
================================================================

PowerShell (Administrador):
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol

Para qué sirve:
Ver versiones de SMB habilitadas.

Interpretación:
- SMBv1 = True  → Riesgo legacy (deshabilitar)
- SMBv2 = True  → Correcto

================================================================
9) VALIDACIÓN DE HOSTS / IP FANTASMA (ARP)
================================================================

Comando:
arp -a | find "192.168.1.X"

Para qué sirve:
Ver si existe relación IP → MAC en caché ARP.

Interpretación:
- Sin salida → host no visto o no existe
- MAC 00:00:00:00:00:00 → valor inválido / IP fantasma

------------------------------------------------

Comando:
arp -d *
ping 192.168.1.X
arp -a | find "192.168.1.X"

Para qué sirve:
Limpiar caché ARP y forzar verificación real.

================================================================
METODOLOGÍA USADA
================================================================

No se buscan errores al azar.
Se responden preguntas concretas:

¿Es Domain Controller?
→ Identidad y DomainRole

¿Está expuesto a internet?
→ Gateway + NAT + puertos

¿Qué servicios escuchan?
→ netstat

¿Quién es dueño del puerto?
→ PID + tasklist

¿Es legítimo?
→ HTTP.sys / IIS / WinRM

¿Hay acceso remoto riesgoso?
→ RDP + WinRM

¿Quién tiene control total?
→ Administrators

¿Hay protocolos legacy?
→ SMB

================================================================
RESULTADO ESPERADO
================================================================

✔️ Arquitectura real confirmada  
✔️ Hipótesis falsas eliminadas  
✔️ Riesgos reales identificados  
✔️ Hardening futuro definido  
✔️ Documentación reutilizable y auditable  

Este flujo aplica a:
- cualquier Windows Server
