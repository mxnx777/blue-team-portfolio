# TryHackMe — Networking Concepts: OSI y TCP/IP aplicados a SOC N1

**Fecha de estudio:** 16-11-2026  
**Ruta:** Cyber Security 101 → Networking Concepts, Task 2 (OSI Model) y Task 3 (TCP/IP Model)  
**Estado comprobado en capturas:** cuatro respuestas correctas sobre OSI; dos respuestas correctas sobre TCP/IP. Siguiente tema: Task 4, IP Addresses and Subnets.

## 0. Objetivo de esta guía

Al releerla debemos poder explicar qué función tiene cada capa, identificar MAC/IP/puerto/protocolo, interpretar por qué falla una conexión y construir hipótesis comprobables sin confundir «no conecta» con «el servidor está caído».

**Fuente de la práctica:** explicaciones y capturas enviadas durante TryHackMe. Las ampliaciones bajo el rótulo «Ejemplo para practicar» son ejercicios didácticos propuestos, NO capturas ni pruebas ejecutadas en esta sesión.

## 1. Siglas completas (aprender repitiendo)

| Sigla | Inglés | Español / Uso |
|---|---|---|
| OSI | Open Systems Interconnection | Interconexión de Sistemas Abiertos; modelo conceptual de 7 capas |
| ISO | International Organization for Standardization | Organización Internacional de Normalización |
| TCP | Transmission Control Protocol | Protocolo de Control de Transmisión; transporte con conexión |
| IP | Internet Protocol | Protocolo de Internet; direccionamiento y encaminamiento |
| UDP | User Datagram Protocol | Protocolo de Datagramas de Usuario; transporte sin conexión |
| MAC | Media Access Control | Control de acceso al medio; dirección de interfaz en enlace |
| ICMP | Internet Control Message Protocol | Protocolo de mensajes de control de Internet |
| HTTP | Hypertext Transfer Protocol | Protocolo de Transferencia de Hipertexto |
| HTTPS | Hypertext Transfer Protocol Secure | HTTP protegido mediante TLS (habitualmente) |
| DNS | Domain Name System | Sistema de Nombres de Dominio |
| DHCP | Dynamic Host Configuration Protocol | Protocolo de Configuración Dinámica de Host |
| SMB | Server Message Block | Compartición de recursos en red |
| SSH | Secure Shell | Protocolo de acceso remoto seguro |
| TLS | Transport Layer Security | Seguridad de la capa de transporte |
| PCAP | Packet Capture | Captura de paquetes |
| SOC | Security Operations Center | Centro de Operaciones de Seguridad |
| RFC | Request for Comments | Publicación de especificaciones/propuestas técnicas de Internet |
| N1 | Nivel 1 | Primer nivel operativo de monitoreo/triage |

## 2. OSI: las siete capas de abajo hacia arriba

El modelo **OSI** es una manera conceptual de organizar la comunicación. No supone que siempre existan siete programas separados.

| Capa | Nombre | Qué resuelve | Qué observar en una investigación |
|---:|---|---|---|
| 1 | Física | Transmisión de bits por medios físicos o radio | Cable roto, enlace sin señal, potencia de radio |
| 2 | Enlace de datos | Entrega de tramas sobre el enlace local; MAC, Ethernet/Wi-Fi | MAC de origen/destino; VLAN; asociación al punto de acceso |
| 3 | Red | Direccionamiento IP y rutas entre redes | IP, máscara, gateway, routers y rutas |
| 4 | Transporte | Entrega extremo a extremo entre aplicaciones; puertos TCP/UDP | SYN/SYN-ACK, rechazo TCP, puertos, retransmisiones |
| 5 | Sesión | Establecimiento, mantenimiento y coordinación de sesiones (modelo OSI) | Estado de sesiones y negociación; varía según protocolo real |
| 6 | Presentación | Codificación, formatos, compresión y cifrado (modelo OSI) | Formato de datos, decodificación, negociación de cifrado |
| 7 | Aplicación | Servicios/protocolos usados por aplicaciones | HTTP, DNS, SMTP, autenticación y mensajes de aplicación |

**Matiz importante:** los protocolos reales no siempre encajan de manera exclusiva en una única capa; en particular las capas OSI 5–7 suelen reunirse en implementaciones modernas.

### 2.1 Ejemplos conectados a la experiencia del laboratorio

- **Capa 1**: un cable cortado impide una conexión por Ethernet aunque usuario y contraseña sean correctos.
- **Capa 2**: un equipo conectado a una VLAN diferente puede quedar aislado aun cuando el switch esté encendido.
- **Capa 3**: un equipo intenta llegar a una IP de otra subred sin ruta o gateway apropiado.
- **Capa 4**: llegamos por IP, pero el intento de conexión TCP al puerto 1433 falla; puede deberse a firewall, ausencia de servicio escuchando u otro motivo.
- **Capa 7**: la conexión TCP se establece, pero SQL Server devuelve un error de inicio de sesión o una consulta falla.

No concluir que un puerto cerrado implica un servidor apagado, ni que un ping exitoso asegura que todos los puertos están abiertos.

### 2.2 MAC vs IP vs puerto (repetición deliberada)

| Campo observado | Ejemplo | Referencia OSI | Para qué sirve |
|---|---|---|---|
| Dirección MAC | `a4:c3:f0:85:ac:2d` (ejemplo de TryHackMe) | Capa 2 | Entrega en enlace local |
| Dirección IP | `192.168.1.50` (ejemplo didáctico) | Capa 3 | Direccionar paquetes entre redes |
| Puerto TCP | `445` | Capa 4 | Destino lógico de un servicio/comunicación |
| Protocolo SMB | SMB / Server Message Block | Capa 7 en visión didáctica | Compartir archivos, impresoras y otros recursos |

**Nota:** una MAC de 48 bits suele expresarse como seis octetos hexadecimales. Los primeros tres octetos suelen contener el prefijo asignado al fabricante, pero existen direcciones locales/aleatorias y suplantación; **MAC no identifica infaliblemente a una persona ni a un fabricante**. Al pasar por un router se reconstruye la trama de enlace para el siguiente salto; la IP puede mantenerse (salvo NAT u otras transformaciones).

### 2.3 Wireshark y captura de tramas

La captura de TryHackMe mostró una trama con dos MAC: destino y origen, después datos de protocolos superiores. Al analizar una captura, mirar:
1. Ethernet origen/destino (capa 2).
2. IP origen/destino (capa 3).
3. TCP/UDP origen/destino y estado (capa 4).
4. Protocolo de aplicación cuando sea visible (por ejemplo DNS o HTTP).

**No inventar** qué procesos generaron el tráfico a partir de una captura de red: esa atribución requiere telemetría de host o correlación.

## 3. TCP/IP: cuatro capas del modelo de TryHackMe

**TCP/IP = Transmission Control Protocol / Internet Protocol.**

En TryHackMe se explicó con cuatro capas (organización recogida en RFC 1122); algunos textos didácticos usan cinco al separar Física.

| TCP/IP (4 capas) | Relación con OSI | Ejemplos de la sala |
|---|---|---|
| Aplicación | OSI 7 + 6 + 5 | HTTP, HTTPS, FTP, POP3, SMTP, IMAP, Telnet, SSH |
| Transporte | OSI 4 | TCP, UDP |
| Internet | OSI 3 | IP, ICMP, IPsec |
| Enlace | OSI 2 y acceso al medio; OSI 1 no se separa como capa autónoma en esta presentación | Ethernet (802.3), Wi-Fi (802.11) |

**Respuestas comprobadas en la Task 3:**
- *To which layer does HTTP belong in the TCP/IP model?* → **Application Layer** ✅.
- *How many layers of the OSI model does the application layer in the TCP/IP model cover?* → **3** (OSI 5, 6, 7) ✅.

### Por qué los modelos no se contradicen

OSI ayuda a **separar funciones** para diagnosticar. TCP/IP agrupa funciones como se suelen describir los protocolos de Internet. A ambos podemos usarlos en una misma investigación:

- «¿Hay ruta IP?» → **Red OSI 3 / Internet TCP/IP**.
- «¿Establece TCP puerto 443?» → **Transporte OSI 4 / Transporte TCP/IP**.
- «¿HTTPS devuelve error 403?» → **Aplicación OSI 7 / Aplicación TCP/IP**.

## 4. Viaje de una petición web: aplicación → transporte → Internet → enlace

Ejemplo didáctico: un navegador pide una página por HTTPS **sobre TCP**.

1. **Aplicación:** navegador inicia intercambio HTTP protegido con TLS (HTTPS).
2. **Transporte:** TCP establece conexión hacia el puerto de destino típico 443 y controla orden/retransmisiones.
3. **Internet:** IP permite transportar los paquetes hacia la red del servidor.
4. **Enlace:** Ethernet o Wi-Fi entrega la trama en el siguiente salto local.

Al recibirlos, el otro lado interpreta las capas correspondientes en sentido inverso. Es una descripción conceptual del **encapsulamiento/desencapsulamiento**.

**Excepción importante:** también existe HTTP/3 sobre QUIC, que utiliza UDP; no afirmar que todo HTTPS siempre usa TCP.

### Ejemplo para practicar A: revisar conectividad de un servicio web

En un equipo Windows autorizado:

~~~powershell
Test-NetConnection example.com -Port 443
~~~

Interpretación: prueba de conexión TCP. Si la prueba falla, puede ser DNS, ruta, filtrado o servicio no escuchando. Si tiene éxito, solo demuestra conectividad TCP, **no** que HTTP/TLS/autenticación sean correctos.

En Linux:

~~~bash
# Solo como comprobación básica de conectividad y negociación HTTP(S):
curl -I https://example.com
~~~

`curl -I` solicita cabeceras (si el servidor lo permite). Si falla, revisar DNS, red, transporte y TLS según el mensaje, antes de concluir causa.

## 5. Caso de diagnóstico conectado a la práctica: SQL Server TCP 1433

En un trabajo anterior se utilizó:

~~~powershell
Test-NetConnection 10.147.17.205 -Port 1433
~~~

La prueba no consiguió conexión TCP desde el equipo que se estaba utilizando. Eso **no demuestra cuál fue la causa**.

### Procedimiento de triage (propuesto, no todas estas pruebas se hicieron allí)

1. **Identificar destino** y confirmar que la IP corresponde al servidor esperado.
2. **Capa 3 / Internet:** revisar IP, máscara, gateway y rutas con `ipconfig /all` y `route print`. Un ping puede aportar una pista, pero ICMP puede estar bloqueado.
3. **Capa 4 / Transporte:** `Test-NetConnection ... -Port 1433` comprueba intento de conexión TCP al puerto.
4. **En el servidor, con permisos administrativos y autorización:** revisar si SQL Server escucha en 1433 y si hay firewall/ACL filtrando. SQL Server puede usar puertos diferentes o instancias dinámicas.
5. **Capa de aplicación:** solo después de tener TCP funcional, comprobar protocolo, cifrado y credenciales de SQL.
6. **Evidencia SOC:** anotar hora, IP origen/destino, puerto, resultado y contexto; evitar atribuir inmediatamente un incidente.

**Error frecuente a evitar:** «Tengo ZeroTier, por lo tanto SQL debe conectar». Una interfaz VPN o red superpuesta funcionando **no garantiza** resolución de ruta, reglas de firewall ni puertos abiertos.

## 6. Caso de análisis SOC: puerto 445 y SMB

Ejemplo de estudio: un registro de tráfico muestra:

~~~text
Destino IP: 192.168.1.50
Destino TCP: 445
~~~

Lectura:
- IP `192.168.1.50` → OSI **3**, TCP/IP **Internet**.
- TCP puerto `445` → OSI **4**, TCP/IP **Transporte**.
- SMB (*Server Message Block*) → protocolo de **aplicación** usado frecuentemente para compartir archivos Windows.

**No es prueba automática de intrusión.** Si un mismo host intenta muchos destinos 445 en poco tiempo, investigar identidad, activos, permisos, eventos y comportamiento; podría ser administración legítima, exploración o actividad maliciosa.

### Ejemplo para practicar B: cruzar puerto con proceso en Windows

~~~powershell
Get-NetTCPConnection -LocalPort 445 -ErrorAction SilentlyContinue |
    Select-Object LocalAddress,LocalPort,State,OwningProcess
~~~

`OwningProcess` es un **PID** (*Process Identifier*, identificador de proceso). Para revisar ese proceso:

~~~powershell
Get-Process -Id <PID>
~~~

Reemplazar `<PID>` por un número real; algunas conexiones/servicios requieren privilegios para consultar todos los datos. La correlación red + proceso mejora el triage pero no prueba por sí sola intención maliciosa.

### Ejemplo para practicar C: DNS vs transporte

~~~powershell
Resolve-DnsName example.com
Test-NetConnection example.com -Port 443
~~~

- `Resolve-DnsName`: comprueba resolución de nombre (**función de aplicación DNS**, aunque involucre transporte/red).
- `Test-NetConnection`: intenta el puerto TCP.
- DNS exitoso + TCP fallido = resolución del nombre aparentemente funciona; sigue pendiente investigar conexión, ruta o filtro.
- TCP exitoso + error HTTP 403 = comunicación TCP funciona pero el servicio de aplicación deniega la solicitud.

## 7. Tabla de diagnóstico «qué sé / qué no sé»

| Observación | Qué puedo afirmar | Qué NO puedo afirmar |
|---|---|---|
| `ping` no responde | No recibí la respuesta ICMP esperada | «El host está apagado» |
| `Test-NetConnection` TCP 1433 = False | No se estableció esa conexión TCP en esa prueba | «La contraseña SQL es incorrecta» |
| TCP 443 = True | Se estableció conexión TCP en ese momento | «La web completa funciona» |
| Wireshark muestra TCP 445 | Existe tráfico orientado al puerto asociado comúnmente a SMB | «Es ransomware» |
| DNS resuelve un nombre | Obtuve resolución DNS en la consulta | «La aplicación remota está sana» |
| MAC de fabricante aparente | El prefijo puede sugerir un fabricante asignado | «Sé quién utilizó ese dispositivo» |

## 8. Preguntas tipo prueba — repaso espaciado

**Ronda A: conceptos y siglas**
1. ¿Qué significan OSI, ISO, TCP e IP?
2. ¿Cuántas capas tiene OSI? ¿Cuántas TCP/IP en TryHackMe?
3. ¿Qué capas OSI agrupa aplicación TCP/IP?
4. ¿Qué significan MAC, DNS, UDP, SMB, SSH e ICMP?
5. ¿Qué diferencia una IP de una MAC?

**Ronda B: identificar capa**
6. Una interfaz sin enlace Ethernet: ¿qué capa mirarías primero?
7. Un switch y direcciones MAC: ¿qué capa?
8. Problema de ruta IP: ¿qué capa?
9. Intento TCP 1433: ¿qué capa?
10. Error de autenticación SQL con TCP funcional: ¿qué nivel de aplicación?
11. ¿HTTP en TCP/IP corresponde a qué capa?
12. ¿Puerto 445 corresponde a qué capa OSI?
13. ¿Cuál es la diferencia entre HTTPS y TCP?
14. ¿Por qué HTTP/3 impide afirmar que todo tráfico HTTPS usa TCP?

**Ronda C: análisis y evidencia**
15. `ping` falla, pero TCP 443 conecta: ¿el host necesariamente está caído?
16. TCP 1433 no conecta: menciona tres hipótesis diferentes y una prueba para cada una.
17. Una conexión SMB 445: ¿es evidencia suficiente de movimiento lateral?
18. ¿Qué información de Wireshark pertenece a las capas 2, 3 y 4?
19. ¿Para qué sirve `OwningProcess` en `Get-NetTCPConnection`?
20. ¿Qué datos guardarías en un informe SOC N1 para que otro analista pueda reproducir una prueba?

### Clave para autocorrección

1. Open Systems Interconnection; International Organization for Standardization; Transmission Control Protocol; Internet Protocol.
2. OSI 7; TCP/IP 4 (presentación de la sala).
3. OSI 5, 6, 7.
4. Media Access Control, Domain Name System, User Datagram Protocol, Server Message Block, Secure Shell, Internet Control Message Protocol.
5. MAC opera en el enlace local; IP direcciona entre redes.
6. Capa 1.
7. Capa 2.
8. Capa 3.
9. Capa 4.
10. Capa de aplicación / servicio.
11. Aplicación.
12. Capa 4.
13. HTTPS es protocolo de aplicación HTTP protegido, TCP es transporte.
14. Porque HTTP/3 usa QUIC sobre UDP.
15. No.
16. Ruta, firewall, servicio no escuchando/puerto diferente (investigar con pruebas de IP/ruta, política de firewall y escucha del servicio).
17. No; requiere correlación.
18. MAC, IP y puertos/protocolo transporte, respectivamente.
19. Asociar conexión con PID.
20. Hora, host, usuario/contexto, IP origen/destino, puerto, comando, salida y limitaciones.

## 9. Resumen de memoria de una sola mirada

**OSI 1 física, 2 enlace/MAC, 3 red/IP, 4 transporte/TCP-UDP/puertos, 5 sesión, 6 presentación, 7 aplicación.**

**TCP/IP aplicación = OSI 5–7; transporte = 4; Internet = 3; enlace = 2 + acceso al medio.**

**En SOC:** no adivinar causas ni ataques solo por un puerto. Formular hipótesis → medir conectividad → revisar servicio → correlacionar registros → documentar.

**Próximo tema de TryHackMe:** IP Addresses and Subnets (direcciones IP y subredes).
