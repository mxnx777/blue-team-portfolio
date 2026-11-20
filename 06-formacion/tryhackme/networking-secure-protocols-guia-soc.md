# TryHackMe — Networking Secure Protocols: TLS, HTTPS, SSH, SFTP/FTPS y VPN

**Fecha:** 20 de noviembre de 2026 · **Serie:** Networking (4 de 4) → *Networking Secure Protocols*
**Entorno:** material de la sala + máquina del desafío final (Wireshark con `randy-chromium.pcapng`)
**Propósito:** entender cómo TLS, SSH y las VPN protegen los protocolos en texto plano de la sala anterior, cómo se ve eso en una captura y qué puede (y no puede) ver un analista SOC.

> Sin flags ni respuestas literales. La contraseña recuperada en el desafío se omite. Las capturas citadas son material de la sala.
> Serie completa: [Core Protocols](networking-core-protocols-guia-soc.md) (sala 3) explica los protocolos en claro que aquí se protegen.

## 1. Diccionario de siglas

| Sigla | Nombre en inglés | Significado |
|---|---|---|
| TLS | Transport Layer Security | Seguridad de la capa de transporte |
| SSL | Secure Sockets Layer | Predecesor de TLS (obsoleto) |
| CA | Certificate Authority | Autoridad de certificación |
| CSR | Certificate Signing Request | Solicitud de firma de certificado |
| SNI | Server Name Indication | Nombre del sitio enviado en el saludo TLS |
| HTTP2 | HyperText Transfer Protocol version 2 | HTTP versión 2 (binario, con flujos) |
| SSH | Secure Shell | Consola remota cifrada |
| SFTP | SSH File Transfer Protocol | Transferencia de archivos dentro de SSH |
| FTPS | FTP Secure | FTP sobre TLS |
| VPN | Virtual Private Network | Red privada virtual |
| IPsec | Internet Protocol Security | Familia de protocolos VPN a nivel IP |
| MFA | Multi-Factor Authentication | Autenticación de múltiples factores |
| CSRF | Cross-Site Request Forgery | Falsificación de petición entre sitios |
| SIEM | Security Information and Event Management | Gestión de eventos e información de seguridad |
| SOC | Security Operations Center | Centro de operaciones de seguridad |

## 2. El problema: qué falta en los protocolos en claro

| Propiedad | Protege contra | Ejemplo |
|---|---|---|
| **Confidencialidad** | Que otro lea | Clave de un buzón POP3 capturada en la red |
| **Integridad** | Que otro modifique | Un pago de 100 alterado a 800 en el camino |
| **Autenticidad** | Hablar con un impostor | Un servidor falso haciéndose pasar por el banco |

TLS agrega las tres **encima** de protocolos existentes, sin cambiar TCP ni IP: HTTP → HTTPS, SMTP → SMTPS, POP3 → POP3S, IMAP → IMAPS. TLS es la evolución de SSL.

## 3. Certificados: el sistema de "sellos"

1. El administrador genera una **CSR** (dominio + clave pública).
2. Una **CA** verifica que controla el dominio y **firma** el certificado.
3. El navegador trae instaladas las CA de confianza (*Certificate Manager → Authorities*). Si el certificado del sitio lo firmó una de ellas, se muestra el candado.

- **Let's Encrypt:** CA gratuita; por eso casi toda la web tiene HTTPS.
- **Certificado autofirmado (*self-signed*):** el servidor se firma a sí mismo. **Cifra, pero no prueba identidad**: nadie lo avaló. Normal en laboratorios (el dashboard de Wazuh lo usa) y sospechoso en un sitio público.

**Lectura SOC:** el candado prueba que hablas con **ese dominio**, no que el dominio sea honesto. Un phishing en un dominio parecido puede tener un certificado válido y gratuito.

## 4. HTTP vs HTTPS en una captura

### HTTP (dos fases)

| Recuadro | Paquetes | Qué ocurre |
|---|---|---|
| 1 | `SYN` → `SYN, ACK` → `ACK` | Mano TCP de tres pasos (*three-way handshake*) |
| 2 | `GET /`, `200 OK`, `GET /favicon.ico`, `404 Not Found` | HTTP legible; el navegador pide solo el ícono de la pestaña |
| 3 | `FIN, ACK` → `FIN, ACK` → `ACK` | Cierre de la conexión TCP |

### HTTPS (tres fases)

| Recuadro | Paquetes | Qué ocurre |
|---|---|---|
| 1 | `SYN`, `SYN-ACK`, `ACK` | Mano TCP |
| 2 | `Client Hello (SNI=tryhackme.com)`, `Server Hello, Change Cipher Spec`, `Change Cipher Spec, Finished` + ACK de TCP | Negociación TLS |
| 3 | `Application Data` | HTTP cifrado; Wireshark no sabe qué es |

| Mensaje TLS | Función |
|---|---|
| Client Hello | El cliente ofrece versiones de TLS y cifrados |
| Server Hello | El servidor elige el cifrado y envía su certificado |
| Change Cipher Spec | "Desde aquí, todo cifrado" |
| Finished | Negociación terminada |

**Cuidado con las definiciones de la pregunta:** para la sala, "negociación y establecimiento de TLS" incluye los recuadros **1 y 2** (la mano TCP es parte del establecimiento). Contar solo el recuadro 2 da un resultado distinto. Hay que leer el enunciado antes de contar.

### Lo que un analista sí ve en HTTPS

- **El SNI viaja en claro** en el Client Hello: el nombre del sitio se lee aunque el resto vaya cifrado. Así un firewall o proxy corporativo filtra por dominio sin descifrar.
- IP de origen y destino, puertos, tamaños, horarios y volumen.

### Lo que se ve con la clave

Con la clave de sesión, los `Application Data` pasan a **HTTP2** legible (`HEADERS[1]: GET /login`, `200 OK`, `DATA`) y aparecen las **cookies de sesión** (`connect.sid=...`). Quien roba una cookie de sesión entra a la cuenta sin la clave (*session hijacking*); TLS existe para que no se vean en el camino.

## 5. Desafío final: descifrar TLS con el *key log* del navegador

**Mecanismo:** el navegador se lanzó con `chromium --ssl-key-log-file=~/ssl-key.log`, que escribe en ese archivo las claves de cada sesión TLS. Wireshark las usa para descifrar la captura.

### Pasos en Wireshark

1. Abrir `randy-chromium.pcapng` (carpeta `Documents`): solo se ve `Application Data`.
2. Clic derecho en un paquete TLS → **Protocol Preferences** → **Transport Layer Security** → **Open Transport Layer Security preferences...**
3. **(Pre)-Master-Secret log filename** → **Browse...** → `Documents/ssl-key.log` → **OK**.

### Encontrar el login

Un formulario de login se **envía** con POST:

```text
http2.headers.method == "POST"
```

Apareció `HEADERS[15]: POST /login/...` con estas cabeceras:

```text
origin: https://www.facebook.com
content-type: application/x-www-form-urlencoded
content-length: 132
```

**Detalle clave de HTTP2:** cabeceras y cuerpo viajan en **tramas separadas** del mismo flujo (*stream*). El `HEADERS` es el sobre; el contenido va en el `DATA` con el mismo número de flujo:

```text
http2.streamid == 15
```

| No. | Dirección | Trama | Qué es |
|---|---|---|---|
| 365 | cliente → servidor | `HEADERS[15]: POST /login` | Sobre del pedido |
| 366 | cliente → servidor | `DATA[15] (application/x-www-form-urlencoded)` | **El formulario con las credenciales** |
| 368 | servidor → cliente | `PING`, `WINDOW_UPDATE` | Control de flujo de HTTP2 |
| 370 | servidor → cliente | `HEADERS[15]: 200 OK` | Respuesta |
| 377–392 | servidor → cliente | `DATA[15]` | Página de respuesta en partes (`text/html` al final) |

En el paquete 366, sección **HTML Form URL Encoded**:

```text
Form item: "email" = "strategos@networking.thm"   ← usuario
Form item: "pass"  = "[omitido]"                  ← contraseña
Form item: "lsd"   = "..."                        ← token anti-CSRF, no es una clave
Form item: "login" = "1"
```

Pestañas inferiores del paquete: `Frame` (bytes cifrados tal como viajaron) → `Decrypted TLS` (descifrados con el key log) → `Decompressed Header` (cabeceras HTTP2 descomprimidas). Que existan las pestañas *Decrypted* prueba que el descifrado funcionó.

### Lecturas SOC del desafío

- **`200 OK` tras un POST de login no significa login exitoso:** muchos sitios responden 200 y muestran el error dentro de la página. Se confirma con el contenido de la respuesta, redirecciones o cookies de sesión nuevas.
- **Herramienta legítima:** así se analiza el tráfico HTTPS de un malware en un *sandbox*.
- **Técnica de ataque:** una variable de entorno `SSLKEYLOGFILE` configurada sin autorización hace que el navegador deje las claves de todas las sesiones de la víctima. Su presencia no explicada es un indicador de compromiso.
- **Credenciales en evidencia:** se reportan y se recomienda el cambio de clave; no se usan ni se copian fuera del caso.

## 6. SSH: el reemplazo de Telnet

- Telnet (puerto **23**) envía todo en claro, incluidas las credenciales. **SSH** (puerto **22**) hace lo mismo cifrado. SSH-1 nació en 1995; **OpenSSH** (1999) es la implementación de código abierto que usa casi todo Linux, Kali y Windows moderno.
- Beneficios: autenticación por clave pública y 2FA, confidencialidad, integridad, **túneles** (*tunneling*) y reenvío gráfico (*X11 forwarding*).

```bash
ssh usuario@IP        # consola remota
ssh IP -X             # permite abrir programas gráficos remotos (ej. Wireshark)
```

- **Huella del servidor (*fingerprint*):** SSH la guarda en la primera conexión. Si cambia, avisa: puede ser una reinstalación legítima o un ataque *man-in-the-middle*. Se verifica antes de aceptar.

**Lectura SOC:** el puerto 22 expuesto a internet recibe fuerza bruta constante. En Wazuh se ve como muchas alertas de autenticación fallida de `sshd` desde la misma IP; el patrón crítico es **muchos fallos seguidos de un éxito** (MITRE ATT&CK **T1110**).

## 7. SFTP vs FTPS

| | SFTP | FTPS |
|---|---|---|
| Qué es | Protocolo distinto que viaja **dentro de SSH** | **FTP + TLS** |
| Puerto | 22 | 990 (implícito) o 21 con `AUTH TLS` |
| Credenciales | Usuario SSH | Usuario FTP + certificado |
| Firewall | Un solo puerto | Sigue usando conexiones de datos aparte |

**Truco:** **S**FTP, la S adelante: primero SSH. FTP**S**, la S al final: FTP con seguridad agregada (como HTTPS).

```bash
sftp usuario@IP       # prompt sftp>: ls, get, put, quit (sin comandos de Linux)
```

**Lectura SOC:** SFTP se ve como tráfico SSH cifrado; no se sabe qué archivos se movieron. Una posible exfiltración se investiga por volumen, horario y destino.

## 8. Tabla de puertos: texto plano → seguro

| Protocolo en claro | Puerto | Versión segura | Puerto |
|---|---|---|---|
| HTTP | 80 | HTTPS | 443 |
| Telnet | 23 | SSH | 22 |
| FTP | 21 | FTPS | 990 |
| SMTP | 25 | Envío con STARTTLS / SMTPS | 587 / 465 |
| POP3 | 110 | POP3S | 995 |
| IMAP | 143 | IMAPS | 993 |

**Trucos de memoria:**
- **21, 22, 23 = Fome, Seguro, Tonto:** FTP, SSH, Telnet.
- Correo seguro empieza con **99**; **IMAPS** termina en 3 igual que IMAP (14**3** → 99**3**); POP3S es el **995**.
- Un STARTTLS anunciado no basta: si el cliente no lo pide, la sesión sigue en claro.

**Lectura SOC:** tráfico de clientes de correo hacia 110 o 143 (y no a 995/993) = credenciales en claro por la red.

## 9. VPN

- **Túnel cifrado** sobre una red insegura: confidencialidad + integridad.
- **Sitio a sitio:** une oficinas completas. **Acceso remoto:** un usuario desde su casa hacia la red de la empresa.
- Tecnologías: IPsec, OpenVPN, WireGuard, VPN SSL/TLS. Variante en malla (*mesh*): cada equipo conecta directo con los otros y un controlador decide quién entra.
- Una VPN comercial **no anonimiza**: cambia en quién se confía (del proveedor de internet al proveedor de la VPN).

**Lecturas SOC:**
- Logs de VPN: quién, desde qué IP y a qué hora. Caso clásico: **viaje imposible** (Santiago 10:00 y otro país 10:15 = credencial probablemente comprometida).
- Los equipos VPN perimetrales son de los más atacados: expuestos por diseño; parcheo prioritario.
- VPN sin MFA = una sola llave para entrar a la red.
- Un túnel que evita el firewall perimetral también evita su registro: punto ciego. Se compensa con aprobación manual de equipos, MFA en el controlador, restricción por puerto en el servidor y baja inmediata de equipos que ya no corresponden.

## 10. Errores en la autoevaluación (cuestionario con trampas)

| Error | Causa | Corrección |
|---|---|---|
| Puerto 23 asociado a FTP (dos veces) y 22 a FTP | Escalera 21-22-23 corrida un lugar | Fome, Seguro, Tonto; repetición en días distintos |
| IMAPS confundido con 995 | Mezcla con POP3S | IMAPS termina en 3 como IMAP |
| Pregunta "¿cuál es INCORRECTA?" respondida con una afirmación verdadera | Lectura rápida del enunciado | Detectar NO / EXCEPTO / INCORRECTA antes de mirar alternativas |
| Contar solo el recuadro 2 como "establecimiento de TLS" | No aplicar la definición dada por la sala | Leer cómo define los términos quien pregunta |

**Patrón:** los errores fueron de memoria de puertos y de lectura de enunciados, no de comprensión. Las preguntas de escenario SOC (certificado autofirmado, SNI, huella SSH, VPN, viaje imposible) se respondieron correctamente.

## 11. Práctica propuesta en BLUELAB (no ejecutada aún)

Replicar el desafío con tráfico propio: descifrar el login HTTPS al dashboard de Wazuh (certificado autofirmado), sin internet. VMs: Kali + WAZUH-SRV01.

```bash
# Kali, terminal 1: el navegador hereda la variable y registra las claves TLS
export SSLKEYLOGFILE=~/ssl-key.log
firefox &

# Kali, terminal 2: captura solo HTTPS hacia el servidor Wazuh
sudo tcpdump -i eth0 -w ~/wazuh-https.pcap host 172.31.250.30 and port 443
```

Luego: iniciar sesión en `https://172.31.250.30`, detener la captura (`Ctrl+C`), cargar `ssl-key.log` en Wireshark y filtrar:

```text
http.request.method == "POST" || http2.headers.method == "POST"
```

Resultado esperado: credenciales del laboratorio visibles tras descifrar. Documentar además cómo detectar una variable `SSLKEYLOGFILE` no autorizada en un endpoint.

## 12. Preguntas de repaso

1. ¿Qué tres propiedades agrega TLS y qué ataque evita cada una?
2. ¿Qué diferencia hay entre un certificado firmado por una CA y uno autofirmado?
3. ¿Qué información sigue visible en una conexión HTTPS sin descifrar?
4. Ordena los mensajes de la negociación TLS y explica cada uno.
5. En HTTP2, ¿por qué el `HEADERS` de un POST no muestra las credenciales y dónde están?
6. ¿Qué indica un `200 OK` después de un POST de login?
7. ¿Qué es `SSLKEYLOGFILE` y por qué sirve tanto al analista como al atacante?
8. La huella SSH de un servidor cambió. ¿Qué haces?
9. Explica la diferencia entre SFTP y FTPS en una frase cada uno.
10. Completa: 21 → __, 22 → __, 23 → __; IMAPS → __, POP3S → __.
11. ¿Qué es un "viaje imposible" y qué revisarías?
12. ¿Por qué una VPN comercial no te hace anónimo?

## 13. Pendientes

- Ejecutar la práctica de la sección 11 y documentarla.
- Lab de fuerza bruta SSH contra WAZUH-SRV01 (Kali + `hydra`), con caso SOC según plantilla y entrada en el catálogo de detecciones.
