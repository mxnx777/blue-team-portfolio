# Guía operativa Blue Team / Active Directory

> Resumen limpio del laboratorio: sólo aprendizaje, errores útiles, lógica, evidencias y conceptos que sirven para portfolio/entrevista.

## Objetivo
Transformar el laboratorio en experiencia demostrable siguiendo:
**hipótesis → prueba → evidencia → interpretación → detección → mitigación → documentación**.

## Topología
| Equipo | Rol | IP |
|---|---|---|
| LAB-DC01 | Domain Controller / Active Directory | 172.31.250.10 |
| LAB-CLIENT01 | Windows 11 + Sysmon + Wazuh Agent | 172.31.250.20 |
| WAZUH-SRV01 | SIEM Wazuh | 172.31.250.30 |
| KALI-ATTACK01 | Máquina atacante | 172.31.250.40 |

## Aprendizajes principales

### 1. Redes antes que SIEM
- Interfaz UP no implica IPv4 correcta.
- IP correcta no implica ruta correcta.
- ARP puede demostrar conectividad local aunque ICMP esté bloqueado.
- Un prefijo incorrecto (/32 vs /24) puede cambiar completamente el alcance.

### 2. Nmap
- `open`: servicio accesible.
- `closed`: host responde, sin listener.
- `filtered`: no hay respuesta suficiente; un firewall puede ocultar el estado real.
- `-Pn`: no depender de ping.
- `-n`: no resolver DNS.

### 3. Puerto → PID → proceso → servicio
Ejemplo real:
`5040 → PID 4516 → svchost.exe → CDPSvc`

Comandos simples:
```text
netstat -ano
tasklist /FI "PID eq 4516"
tasklist /svc /FI "PID eq 4516"
```

### 4. Firewall probado con evidencia
- Windows: 445 LISTENING / PID 4.
- Kali: 445 filtered.
- `pfirewall.log`: DROP TCP desde 172.31.250.40 hacia 172.31.250.20:445.
- Lección: aislar origen + destino + protocolo + puerto + hora.

### 5. SMB — Server Message Block
- Enumeración anónima rechazada.
- SMBv1 no observado.
- SMB2/SMB3 presentes.
- SMB signing: enabled and required.
- Puerto abierto no equivale a vulnerabilidad.

### 6. Identidades y mínimo privilegio
- Cuenta local ≠ cuenta de dominio.
- AD — Active Directory centraliza cuentas/grupos/equipos del dominio.
- Una cuenta estándar puede leer bastante contexto del directorio.
- Una cuenta administrativa debe estar separada y pertenecer sólo a los grupos necesarios.

### 7. Eventos de autenticación
- 4625 = Failed Logon.
- 4624 = Successful Logon.
- Logon Type 3 = logon por red.
- `0xC000006A` = usuario correcto, contraseña incorrecta.

### 8. LDAP — Lightweight Directory Access Protocol
LDAP es el protocolo usado para consultar directorios como Active Directory.

Modelo mental:
`quiero saber usuarios → LDAP → filtro de usuario → atributo a mostrar`

Una cuenta estándar autenticada permitió descubrir:
- usuarios;
- grupos;
- equipos;
- membresías;
- estructura del dominio.

### 9. DN — Distinguished Name
Ejemplo:
`CN=adm-mxnx,OU=ADMINISTRADORES,OU=USUARIOS,OU=BLUELAB,DC=bluelab,DC=test`

- CN = Common Name
- OU = Organizational Unit
- DC = Domain Component

### 10. SPN / Kerberos
- SPN = Service Principal Name.
- TGS = Ticket Granting Service.
- Una cuenta de servicio con clave débil puede ser un objetivo importante.
- Event ID 4769 permite investigar solicitudes de tickets de servicio.
- La defensa debe mirar identidad, servicio, origen, hora y cifrado.

## Siglas que hay que repetir
- **SOC** — Security Operations Center.
- **SIEM** — Security Information and Event Management.
- **AD** — Active Directory.
- **LDAP** — Lightweight Directory Access Protocol.
- **SMB** — Server Message Block.
- **ARP** — Address Resolution Protocol.
- **DNS** — Domain Name System.
- **EDR** — Endpoint Detection and Response.
- **XDR** — Extended Detection and Response.
- **SPN** — Service Principal Name.
- **TGS** — Ticket Granting Service.
- **MITRE ATT&CK** — Adversarial Tactics, Techniques and Common Knowledge.

## Errores útiles
| Error | Lección |
|---|---|
| /32 en vez de /24 | IP, máscara y ruta deben analizarse juntas |
| IP temporal con `ip addr add` | NetworkManager puede reemplazarla |
| `-pn` vs `-Pn` | las opciones distinguen mayúsculas |
| buscar regla por nombre inglés | preferir campos técnicos no localizados |
| `filtered` = cerrado | falso; hay que cruzar con el endpoint |
| sintaxis SMB mezclada | separar dominio/usuario |
| LDAP mal partido en líneas | leer la salida antes de culpar credenciales |
| asumir privilegios por nombre | comprobar grupos/permisos |

## Plantilla GitHub para futuros casos
1. Objetivo
2. Alcance/topología
3. Hipótesis
4. Prueba
5. Evidencia
6. Interpretación
7. MITRE ATT&CK
8. Mitigación
9. Resultado
10. Qué aprendí / qué explicaría en entrevista
