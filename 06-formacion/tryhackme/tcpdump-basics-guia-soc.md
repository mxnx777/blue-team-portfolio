# TryHackMe — Tcpdump: The Basics

**Fecha:** 23–24 de noviembre de 2026 · **Ruta:** Cyber Security 101 → Tcpdump: The Basics
**Entorno:** máquina de la sala (terminal en el navegador) con `traffic.pcap`
**Propósito:** capturar, filtrar y leer tráfico desde la consola, como se hace en un servidor sin interfaz gráfica, con lectura SOC de lo encontrado.

> Sin flags ni respuestas literales de la sala: se omiten conteos, IP, MAC y nombres que piden las preguntas.
> Relacionado: [lab de ICMP, TTL y ARP](../fundamentos/redes-icmp-ttl-arp-lab.md) (primer uso de tcpdump en BLUELAB) y [Wireshark: The Basics](wireshark-basics-guia-soc.md).

## 1. Diccionario de siglas

| Sigla | Nombre en inglés | Significado |
|---|---|---|
| libpcap | Packet Capture library | Librería de captura usada por tcpdump, Wireshark, tshark y nmap (en Windows: winpcap, hoy npcap) |
| PCAP | Packet Capture | Archivo de captura |
| ARP | Address Resolution Protocol | Resolución IP → MAC en la red local |
| ICMP | Internet Control Message Protocol | Ping, traceroute |
| STP | Spanning Tree Protocol | Protocolo de switches para evitar bucles (ruido normal) |
| OUI | Organizationally Unique Identifier | Primera mitad de la MAC: identifica al fabricante |
| SOC | Security Operations Center | Centro de operaciones de seguridad |

## 2. Opciones básicas

| Opción | Qué hace |
|---|---|
| `-i INTERFAZ` | Escuchar en una interfaz (`eth0`, `ens5`; `any` = todas). Ver interfaces: `ip a s` |
| `-w ARCHIVO` | **W**rite: guarda en `.pcap` para abrir después en Wireshark |
| `-r ARCHIVO` | **R**ead: lee una captura (no requiere `sudo`) |
| `-c N` | Se detiene tras N paquetes |
| `-n` | No resuelve IP a nombres (sin consultas DNS) |
| `-nn` | No resuelve IP ni puertos (`80` en vez de `http`) |
| `-v`, `-vv`, `-vvv` | Más detalle (TTL, longitudes, opciones) |

**Errores propios:**
- `tcpdump -r traffic.pcap -n-e arp` → `invalid option -- '-'`. Las opciones van separadas (`-n -e`) o juntas con un solo guion (`-ne`), nunca `-n-e`.
- `SSH user@...` → `command not found`: Linux distingue mayúsculas; los comandos van en minúscula.
- Se buscó `traffic.pcap` desde la AttackBox, donde no existía; el archivo estaba en la máquina de la sala. **Leer el prompt** (`usuario@equipo`) antes de cada comando.
- Se hizo SSH desde la máquina hacia sí misma: el log mostró `Last login ... from` su propia IP.

## 3. Leer una línea de tcpdump

```text
08:55:18.989213 IP 10.10.117.2.22 > 10.11.81.126.53378: Flags [P.], seq ..., length 196
```

| Parte | Significado |
|---|---|
| `10.10.117.2.22` | IP y **puerto pegados con un punto** (puerto 22 = SSH) |
| `>` | Lo de antes **envía**, lo de después **recibe** |
| `Flags [P.]` | P = PSH, `.` = ACK |
| `length 196` | Bytes de datos |

El ejemplo de la sala capturaba **su propia sesión SSH** (puerto 22): cada línea mostrada generaba más tráfico SSH. Para evitarlo: `not port 22`.

## 4. Filtros de captura

| Tipo | Ejemplos |
|---|---|
| Host | `host IP`, `src host IP`, `dst host IP` |
| Puerto | `port 53`, `src port N`, `dst port N` |
| Protocolo | `icmp`, `arp`, `tcp`, `udp`, `ip`, `ip6` |
| Tamaño | `greater 15000`, `less 60` (en palabras, nunca `>` ni `<`: en la terminal `>` redirige a un archivo) |
| Lógicos | `and`, `or`, `not` |

```bash
tcpdump -r traffic.pcap -n icmp | wc -l                     # contar paquetes ICMP
tcpdump -r traffic.pcap -n arp | grep "who-has <IP>"        # quién pregunta por una IP
tcpdump -r traffic.pcap -n port 53 -c 1                     # primera consulta DNS
```

- `|` (tubería) pasa la salida al siguiente comando; `wc -l` cuenta líneas = paquetes; `grep` filtra líneas por texto.
- La línea `reading from file...` va por el canal de errores y no se cuenta.
- En ARP, **el que pregunta es el que aparece después de `tell`**.
- En DNS: `47108+ A? example.org.` = ID de transacción, recursión pedida, tipo de registro y nombre. El punto final del nombre no forma parte del dominio.

**Filtro de captura = tijera** (lo descartado no se graba). **Filtro de visualización = lupa** (solo oculta).

## 5. Banderas TCP

| Bandera | Significado |
|---|---|
| SYN | Synchronize: inicio del handshake |
| ACK | Acknowledge: recibido |
| FIN | Finish: cierre ordenado |
| RST | Reset: cierre brusco / "aquí no hay nadie" |
| PSH | Push: entregar los datos ya |

```bash
tcpdump -r traffic.pcap -n "tcp[tcpflags] == tcp-rst"        # SOLO RST, nada más
tcpdump -r traffic.pcap -n "tcp[tcpflags] & tcp-rst != 0"    # RST, con o sin otras banderas
```

- **`==` = "solo esto"**; **`&` ... `!= 0` = "por lo menos esto"**. La primera cuenta igual o menos que la segunda.
- Comillas obligatorias cuando el filtro tiene corchetes o símbolos.
- Fue la parte que más costó; se resolvió con la analogía de los interruptores (todos los demás abajo vs. "los demás como sea").

**Lectura SOC:** muchos RST desde un host = respuestas de puertos cerrados ante un **escaneo de puertos**.

## 6. Mostrar paquetes

| Opción | Muestra |
|---|---|
| `-q` | Salida breve |
| `-e` | Agrega las MAC |
| `-A` | Contenido en ASCII (solo legible si el protocolo va en claro; HTTPS se ve como basura) |
| `-xx` | Contenido en hexadecimal |
| `-X` | Hexadecimal + ASCII (como el panel de bytes de Wireshark) |

Con `-e`, cada línea es `MAC_ORIGEN > MAC_DESTINO`: **la primera MAC es la del emisor de ese paquete**. En un par ARP, el Request trae la MAC de quien pregunta y el Reply la de quien responde; hay que elegir la línea según lo que se busca.

## 7. Perfil de un host armado desde la captura

Con pocas consultas a `traffic.pcap` se reunió sobre el host `.137`:
- Recibe SSH (puerto 22).
- Es el objetivo de una consulta ARP.
- Consulta DNS por los espejos de una distribución Linux → hipótesis: servidor buscando actualizaciones.
- Recibe paquetes de más de 15.000 bytes desde un servidor web (puerto 80) → descarga grande.
- MAC con prefijo `52:54:00` → máquina virtual QEMU/KVM.

Nota técnica: paquetes de más de 1.500 bytes en una captura tomada en el propio equipo suelen deberse al **offloading** (la tarjeta junta segmentos antes de pasarlos al sistema). No son un ataque.

## 8. Lecturas SOC

- tcpdump es la herramienta para **servidores sin escritorio** y sesiones SSH: se captura con `-w` y se analiza después en Wireshark.
- Excluir la propia sesión con `not port 22` al capturar por SSH.
- **ARP spoofing:** una misma IP con dos MAC distintas, o respuestas ARP que nadie pidió. Se detecta con `-e`.
- Un login SSH "desde sí mismo" en los logs merece una segunda mirada.

## 9. Preguntas de repaso

1. ¿Qué librería usan tcpdump, Wireshark y nmap para capturar?
2. ¿Por qué leer un `.pcap` con `-r` no requiere `sudo`?
3. ¿Qué diferencia hay entre `-n` y `-nn`?
4. ¿Por qué se escribe `greater 15000` y no `> 15000`?
5. ¿Qué hace `| wc -l` y por qué sirve para contar paquetes?
6. En un ARP Request, ¿cómo identificas quién pregunta?
7. Diferencia entre `"tcp[tcpflags] == tcp-rst"` y `"tcp[tcpflags] & tcp-rst != 0"`.
8. ¿Qué sugiere ver muchos RST desde un mismo host?
9. ¿Por qué `-A` no sirve para leer HTTPS?
10. ¿Cómo detectarías un ARP spoofing con tcpdump?
11. ¿Por qué conviene `not port 22` al capturar por SSH?

<details>
<summary><strong>Respuestas (abrir después de intentarlo)</strong></summary>

1. **libpcap**. En Windows su equivalente es npcap (antes winpcap).
2. Porque leer un archivo no toca la tarjeta de red. Capturar en vivo sí requiere root.
3. `-n` no traduce IP a nombres; `-nn` tampoco traduce puertos. Ej.: con `-n` ves `10.0.0.5.http`; con `-nn`, `10.0.0.5.80`.
4. En la terminal `>` redirige la salida a un archivo: `tcpdump ... > 15000` crearía un archivo llamado `15000` en vez de filtrar.
5. Cuenta líneas, y tcpdump muestra un paquete por línea. Ej.: `tcpdump -r cap.pcap -n icmp | wc -l` → `12` = 12 paquetes ICMP.
6. Es la IP que aparece después de `tell`. Ej.: `Request who-has 172.31.250.20 tell 172.31.250.40` → pregunta la `.40`.
7. `==` cuenta paquetes con **solo** RST; `&` ... `!= 0` cuenta también los que traen RST junto con otra bandera (por ejemplo RST+ACK). El segundo da igual o más.
8. Un escaneo de puertos: el host responde RST desde cada puerto cerrado que le tocan.
9. Porque HTTPS va cifrado: en ASCII sale basura como `E..M..@.5..)h...`. `-A` solo sirve con protocolos en claro (HTTP, FTP, Telnet).
10. Con `-e`: la misma IP aparece con **dos MAC distintas**, o llegan respuestas `is-at` que nadie pidió.
11. Porque tu propia sesión SSH genera tráfico que tcpdump muestra, eso genera más tráfico SSH, y así sin fin: tapa lo que buscas.

</details>
