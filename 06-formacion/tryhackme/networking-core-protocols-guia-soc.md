# TryHackMe — Networking Core Protocols: DNS, WHOIS, HTTP, FTP, SMTP, POP3 e IMAP

**Fecha:** 19 de noviembre de 2026 · **Serie:** Networking (3 de 4) → *Networking Core Protocols*
**Entorno:** AttackBox de TryHackMe + máquina objetivo de la sala
**Propósito:** entender cómo se ven por dentro los protocolos de aplicación más comunes, hablándolos a mano con `telnet`, y conectarlos con análisis Blue Team / SOC N1.

> Sin flags ni respuestas literales de la sala. Las IP de la máquina objetivo y de la AttackBox son temporales de TryHackMe. Las capturas DNS de la sección 2 son tráfico real observado en la AttackBox, no material de la sala.

## 1. Diccionario de siglas

| Sigla | Nombre en inglés | Significado |
|---|---|---|
| DNS | Domain Name System | Sistema de nombres de dominio |
| TCP / UDP | Transmission Control Protocol / User Datagram Protocol | Transporte con conexión / sin conexión |
| HTTP / HTTPS | HyperText Transfer Protocol (Secure) | Protocolo de transferencia de hipertexto (seguro) |
| FTP | File Transfer Protocol | Protocolo de transferencia de archivos |
| SFTP / FTPS | SSH File Transfer Protocol / FTP Secure | Transferencia de archivos cifrada |
| SMTP / ESMTP | (Extended) Simple Mail Transfer Protocol | Protocolo simple de transferencia de correo (extendido) |
| POP3 | Post Office Protocol version 3 | Protocolo de oficina de correos, versión 3 |
| IMAP | Internet Message Access Protocol | Protocolo de acceso a mensajes de internet |
| TLS | Transport Layer Security | Seguridad de la capa de transporte (cifrado) |
| SSH | Secure Shell | Consola remota cifrada |
| SPF | Sender Policy Framework | Qué IP pueden enviar correo por un dominio |
| DKIM | DomainKeys Identified Mail | Firma criptográfica del correo |
| DMARC | Domain-based Message Authentication, Reporting and Conformance | Política ante fallos de SPF/DKIM |
| A / AAAA | Address | Registro DNS de IPv4 / IPv6 |
| CNAME | Canonical Name | Registro DNS de alias |
| MX | Mail Exchange | Registro DNS del servidor de correo |
| NS | Name Server | Servidor DNS responsable de un dominio |
| SOA | Start of Authority | Registro de autoridad de una zona DNS |
| CDN | Content Delivery Network | Red de distribución de contenido |
| GDPR | General Data Protection Regulation | Reglamento europeo de protección de datos |
| C2 | Command and Control | Servidor de control de un atacante |
| SOC | Security Operations Center | Centro de operaciones de seguridad |

## 2. Resumen de puertos

| Protocolo | Puerto por defecto | Versión cifrada | ¿Texto plano? |
|---|---|---|---|
| DNS | 53 UDP (TCP como alternativa) | DoT 853 / DoH 443 | Sí |
| WHOIS | 43 TCP | — | Sí |
| HTTP / HTTPS | 80 / 443 TCP | HTTPS | HTTP sí |
| FTP | 21 TCP (control) + puertos de datos | SFTP (22) / FTPS | Sí, incluidas credenciales |
| SMTP | 25 TCP (587 para envío autenticado) | STARTTLS / 465 | Sí sin TLS |
| POP3 | 110 TCP | 995 | Sí sin TLS |
| IMAP | 143 TCP | 993 | Sí sin TLS |
| Telnet | 23 TCP | SSH (22) | Sí, incluidas credenciales |

**Idea central de la sala:** todos estos protocolos son **texto**. Un cliente manda líneas legibles y el servidor responde con líneas legibles. Sin TLS, cualquiera que capture el tráfico lee todo.

## 3. DNS: nombres a direcciones

### Registros vistos

| Registro | Pregunta que responde |
|---|---|
| A | ¿Qué IPv4 tiene este nombre? (uno o varios) |
| AAAA | ¿Qué IPv6 tiene este nombre? |
| CNAME | ¿Este nombre es alias de otro? |
| MX | ¿Qué servidor recibe el correo del dominio? |

Truco de memoria: IPv4 = 32 bits, IPv6 = 128 bits = 4 × 32. **Una A por cada 32 bits.**

### `nslookup`

```bash
nslookup www.example.com
```

- `Server: 127.0.0.1` (o `127.0.0.53`): resolver **local** del sistema, que a su vez consulta al DNS real.
- `Non-authoritative answer`: la respuesta viene de un servidor intermedio o de caché, no del dueño del dominio.
- Aparecieron **varias** IPv4 y IPv6: el dominio está detrás de una CDN que reparte carga.

### Captura propia con `tshark`

El archivo `dns-query.pcapng` del ejemplo de la sala **no existe** en la AttackBox (`-r` lee una captura guardada). Se capturó en vivo:

```bash
tshark -i any -f "port 53" -c 8
```

- `-i any`: todas las interfaces.
- `-f "port 53"`: filtro de captura, solo DNS.
- `-c 8`: se detiene a los 8 paquetes.

Para capturar una consulta propia se necesita dejar `tshark` en segundo plano y darle tiempo a arrancar:

```bash
tshark -i any -f "port 53" -c 20 &
sleep 2; nslookup www.example.com
```

**Lo observado (sin haber hecho ninguna consulta):**

- La AttackBox consultaba por sí sola el mismo dominio cada ~5 segundos (A + AAAA), con IDs de transacción nuevos en cada ronda.
- **Respuestas desordenadas:** la respuesta AAAA llegó antes que la A. El **ID de transacción** (`0x....`) es lo que permite emparejarlas.
- **Cadena CNAME:** el nombre consultado era alias de un balanceador de carga de AWS (`elb.eu-west-1.amazonaws.com`), que devolvía dos IPv4 privadas `172.31.x.x`, rotando el orden (*round-robin*).
- **AAAA sin IPv6:** la respuesta traía un **SOA** en vez de una dirección; es la forma de DNS de decir "no existe ese registro".
- **Dos conversaciones:** `127.0.0.1 → 127.0.0.1` (programa → resolver local) y, cuando el nombre no estaba en caché, `IP de la AttackBox → 169.254.169.253` (resolver local → DNS interno de AWS).

### Lectura SOC

- Consultas periódicas al mismo dominio con intervalo regular = patrón de **beacon**. Aquí era legítimo (la propia AttackBox), pero es el mismo patrón de un malware hablando con su C2 por DNS (MITRE ATT&CK **T1071.004**). Se distingue por reputación del dominio, proceso que consulta y regularidad del intervalo.
- En Windows, **Sysmon Event ID 22** registra cada consulta DNS con el proceso que la hizo.

## 4. WHOIS: quién registró un dominio

```bash
whois dominio.com
```

Quien **registra** un dominio (pago anual a un *registrar*) obtiene la autoridad para crear sus registros DNS. Sus datos de contacto quedan en la base WHOIS (puerto 43). WHOIS no es sigla: viene de *who is*.

| Campo | Para qué sirve |
|---|---|
| `Creation Date` | Antigüedad del dominio |
| `Updated Date` / `Expiration Date` | Última modificación / vencimiento |
| `Registrar` | Empresa con la que se registró |
| `Registrar Abuse Contact` | A dónde reportar abuso |
| `Registrant ...` | Dueño; hoy a menudo oculto por servicios de privacidad (*Domains By Proxy*) o por GDPR (*REDACTED FOR PRIVACY*) |
| `Domain Status: client...Prohibited` | Candados puestos por el registrar |
| `Domain Status: server...Prohibited` | Candados puestos por el registro (*registry lock*), típico de marcas grandes |
| `Name Server` | Servidores DNS que responden por el dominio |

**Lectura SOC (phishing):** un `Creation Date` de hace pocos días es bandera roja. Un registrante oculto, por sí solo, no prueba nada.

## 5. HTTP: hablarle a un servidor web a mano

```bash
telnet <IP> 80
```

```text
GET / HTTP/1.1
Host: anything

```

- `telnet <IP> 80`: conexión TCP cruda al puerto 80; lo que se escribe se envía tal cual. No aparece prompt.
- `GET /`: pide la raíz del sitio. Para otro archivo: `GET /archivo.html HTTP/1.1`.
- `Host:`: qué sitio se pide (un servidor puede alojar varios en la misma IP).
- **Línea vacía final obligatoria:** indica fin del pedido.
- Salir de telnet: `Ctrl + ]` y luego `quit`.

| Método | Uso |
|---|---|
| GET | Obtener un recurso |
| POST | Enviar datos (formularios, login) |
| PUT | Crear o reemplazar un recurso |
| DELETE | Borrar un recurso |

### Respuesta

| Parte | Significado |
|---|---|
| `HTTP/1.1 200 OK` | Código de estado: 2xx bien, 3xx redirección, 4xx error del cliente (404 no existe, 403 prohibido), 5xx error del servidor |
| `Server: nginx/1.18.0 (Ubuntu)` | Software **y versión**: fuga de información |
| `Content-Length` | Tamaño del cuerpo en bytes |
| `Last-Modified` / `ETag` | Fecha de modificación / huella del archivo para caché |
| `Connection: keep-alive` | La conexión queda abierta para más pedidos (y se cierra sola por inactividad: `Connection closed by foreign host`) |
| Línea vacía | Separa cabeceras de cuerpo |

### Lo aprendido

- El contenido escondido en la sala estaba en el HTML con **texto blanco sobre fondo blanco y tamaño 1 px**: invisible en el navegador, visible en el código fuente. Los correos de phishing usan el mismo truco para engañar filtros. **El analista mira el código crudo, no la vista renderizada.**
- Hardening: ocultar versión (`server_tokens off;` en nginx).
- Firefox marca la página como *Not Secure*: HTTP sin cifrar.

## 6. FTP: dos conexiones y credenciales en claro

```bash
ftp <IP>
```

| Respuesta | Significado |
|---|---|
| `220 (vsFTPd 3.0.5)` | Saludo con software y versión |
| `331 Please specify the password.` | Usuario recibido, falta clave |
| `230 Login successful.` | Dentro (con usuario `anonymous`) |
| `229 Entering Extended Passive Mode (\|\|\|puerto\|)` | Se abre un puerto aparte para datos |
| `150` / `226` | Inicio / fin de transferencia |
| `221 Goodbye.` | Cierre |

Comandos del cliente: `ls` (se traduce a `LIST`), `get archivo` (`RETR archivo`), `quit`.

**Dos conexiones:** el puerto 21 lleva los comandos; cada listado o archivo viaja por una conexión de datos nueva. Se vio un puerto pasivo distinto en cada transferencia.

**Hallazgos de auditoría:** FTP anónimo habilitado; `USER` y `PASS` legibles en una captura. Reemplazo: SFTP o FTPS.

## 7. SMTP: enviar correo (y por qué existe el spoofing)

```bash
telnet <IP> 25
```

| Línea | Significado |
|---|---|
| `220 ... ESMTP Exim 4.95` | Saludo con software |
| `HELO client.thm` → `250 ... [IP]` | Saludo; el servidor **anota la IP real** |
| `MAIL FROM: <...>` | Remitente del **sobre** (*envelope*) |
| `RCPT TO: <...>` | Destinatario |
| `DATA` → `354` | Inicio del contenido |
| `From:` / `To:` / `Subject:`, línea vacía, cuerpo | El correo (cabeceras + cuerpo) |
| `.` sola en una línea | Fin del mensaje |
| `250 OK id=...` | Aceptado, con ID de mensaje para rastrear en logs |
| `QUIT` → `221` | Cierre |

**SMTP no verifica identidad.** El `MAIL FROM` (sobre) y el `From:` (lo que ve el usuario) pueden ser distintos e inventados. Las defensas se agregaron encima vía DNS: **SPF**, **DKIM** y **DMARC**.

**Por qué no es anónimo ni simple de abusar:** la IP real queda en las cabeceras `Received:`; los proveedores residenciales bloquean la salida al puerto 25; los servidores no reenvían a dominios ajenos (*open relay*) y los grandes proveedores validan SPF/DKIM/DMARC. El spam real usa botnets, cuentas robadas (que **sí** pasan SPF/DKIM), dominios parecidos recién registrados o servicios masivos contratados con datos falsos.

## 8. POP3: bajar el correo

```bash
telnet <IP> 110
```

Responde `+OK` / `-ERR`, sin códigos numéricos.

| Comando | Qué hace |
|---|---|
| `USER <usuario>` / `PASS <clave>` | Autenticación en texto plano |
| `STAT` | Cantidad y tamaño total de mensajes |
| `LIST` | Mensajes y tamaño de cada uno |
| `RETR <n>` | Trae el mensaje n (`+OK 454 octets`: octeto = byte) |
| `QUIT` | Cierra sesión |

### Cabecera `Received` de un mensaje recuperado

```text
Received: from [10.11.81.126] (helo=client.thm)
        by example.thm with smtp (Exim 4.95)
        (envelope-from <user@client.thm>)
        id 1sMrpq-0001Ah-UT
```

- `from [IP]`: IP real de origen (no falsificable por el remitente).
- `helo=`: nombre que el cliente **dijo** tener (sí falsificable).
- `envelope-from`: el `MAIL FROM` del sobre.
- `id`: el mismo ID que entregó SMTP al aceptar el correo.

Dos correos distintos con la misma IP y el mismo `helo` en `Received:` = mismo origen. Así se correlaciona una campaña de phishing.

## 9. IMAP: sincronizar el correo

```bash
telnet <IP> 143
```

POP3 baja y normalmente borra del servidor; **IMAP deja el correo en el servidor y sincroniza** estado (leído, movido, borrado) entre dispositivos.

**Etiquetas (*tags*):** cada comando empieza con una etiqueta elegida por el cliente (`A`, `B`, `a001`...). El servidor responde con la misma etiqueta; las líneas `*` son datos sin etiqueta. Misma idea que el ID de transacción DNS: emparejar pedido y respuesta, necesario porque IMAP admite varios comandos simultáneos. La etiqueta **no es parte del comando**.

| Comando | Qué hace |
|---|---|
| `A LOGIN <usuario> <clave>` | Autenticación |
| `B SELECT inbox` | Abre una carpeta (`* n EXISTS`, `UNSEEN`, `FLAGS`) |
| `C FETCH <n> body[]` | Trae cabeceras y cuerpo del mensaje n |
| `MOVE` / `COPY` | Mueve / copia mensajes entre carpetas |
| `D LOGOUT` | Cierra sesión |

- `FLAGS (\Answered \Flagged \Deleted \Seen \Draft)`: las marcas que se sincronizan.
- `STARTTLS`: el servidor ofrece pasar a cifrado; si el cliente no lo pide, todo sigue en claro. `AUTH=PLAIN`: acepta credenciales en texto plano.
- El mismo software (Dovecot) atendía POP3 e IMAP.

**Lectura SOC:** en una cuenta comprometida, los logs de IMAP muestran `LOGIN` desde IP y horarios anómalos, seguidos de muchos `FETCH` (descarga masiva del buzón).

## 10. Errores propios y cómo se resolvieron

| Error | Causa | Solución |
|---|---|---|
| `tshark -r dns-query.pcapng` → *file doesn't exist* | El archivo solo existía en el ejemplo de la sala | Capturar en vivo con `-i any -f "port 53"` |
| La consulta propia no apareció en la captura | `tshark` tarda en arrancar y `-c 8` se llenó con tráfico de fondo | `&` + `sleep 2` + `-c` más alto |
| `cat flag.txt` dentro de `ftp>` → *Invalid command* | Dentro del cliente FTP no hay comandos de Linux | `get` → `quit` → `cat` en la terminal |
| `telnet <IP>` sin puerto → consola de Linux del objetivo | Sin puerto, telnet usa el 23 (inicio de sesión remoto) | `exit` y repetir con `telnet <IP> 110` |
| `RETR 4` → `-ERR Noise after message number` | Carácter invisible: Backspace enviado por telnet o espacio final | Escribir la línea completa sin borrar |
| Nueva IP del objetivo a mitad de la sala | La máquina se reinició | Verificar siempre la IP actual antes de conectar |

**Lección transversal:** saber siempre en qué prompt se está (`root@attackbox#`, `linda@objetivo$`, `ftp>`, sesión telnet sin prompt).

## 11. Patrones que se repiten en todos los protocolos

1. **El saludo (*banner*) delata software y versión:** nginx, vsFTPd, Exim, Dovecot. Insumo para un atacante; hardening para el defensor.
2. **Pedido y respuesta se emparejan:** `seq` en ICMP, ID de transacción en DNS, etiquetas en IMAP.
3. **Códigos de estado:** 2xx éxito, 3xx continuar/redirigir, 4xx–5xx error (HTTP, FTP, SMTP); `+OK`/`-ERR` en POP3; `OK`/`NO`/`BAD` en IMAP.
4. **Texto plano por defecto:** credenciales legibles en FTP, Telnet, POP3, IMAP y SMTP sin TLS.
5. **Patrones repetitivos delatan programas:** beacon DNS cada pocos segundos, como el ARP repetido del agente Wazuh en el [lab de redes](../fundamentos/redes-icmp-ttl-arp-lab.md).

## 12. Preguntas de repaso

1. ¿Por qué DNS usa UDP por defecto y cuándo pasa a TCP?
2. Un `nslookup` devuelve cuatro IPv4 distintas para el mismo nombre. ¿Qué explica eso?
3. ¿Para qué sirve el ID de transacción DNS y qué demostró la captura sobre el orden de las respuestas?
4. ¿Qué indica una respuesta AAAA que trae un SOA y ninguna dirección?
5. Recibes un correo de un dominio con `Creation Date` de hace tres días. ¿Qué concluyes y qué más revisas?
6. ¿Por qué la línea vacía es obligatoria al escribir un pedido HTTP a mano?
7. ¿Qué información entrega `Server: nginx/1.18.0 (Ubuntu)` y cómo se mitiga?
8. ¿Por qué FTP necesita dos conexiones?
9. Explica la diferencia entre `MAIL FROM` y `From:` y por qué permite el spoofing.
10. ¿Qué campo de una cabecera `Received:` no puede falsificar el remitente?
11. ¿Qué diferencia funcional hay entre POP3 e IMAP?
12. ¿Para qué sirven las etiquetas en IMAP?
13. ¿Por qué un puerto 23 abierto es un hallazgo de auditoría?
14. ¿Qué tienen en común un beacon DNS cada 5 s y un ARP repetido sin respuesta?

## 13. Pendientes

- Repetir el análisis DNS en BLUELAB: consultas desde LAB-CLIENT01 hacia DC01, capturadas con `tcpdump` y registradas por **Sysmon Event ID 22** en Wazuh.
- Sala 4 de la serie: *Networking Secure Protocols* (TLS, HTTPS, SSH, SFTP/FTPS).
