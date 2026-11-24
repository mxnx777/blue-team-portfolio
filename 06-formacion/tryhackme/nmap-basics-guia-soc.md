# TryHackMe — Nmap: The Basics

**Fecha:** 24 de noviembre de 2026 · **Ruta:** Cyber Security 101 → Nmap: The Basics
**Entorno:** AttackBox + máquina objetivo de la sala
**Propósito:** descubrir hosts, escanear puertos, detectar servicios y sistema operativo, controlar la velocidad y guardar resultados, entendiendo qué huella deja cada técnica para el defensor.

> Sin flags ni respuestas literales de la sala: se omiten la bandera, conteos y versiones que piden las preguntas.
> Relacionado: [Tcpdump: The Basics](tcpdump-basics-guia-soc.md) y [lab de ICMP, TTL y ARP](../fundamentos/redes-icmp-ttl-arp-lab.md).

## 1. Diccionario de siglas

| Sigla | Nombre en inglés | Significado |
|---|---|---|
| CIDR | Classless Inter-Domain Routing | Notación de prefijo (`/24`, `/27`) |
| ACL | Access Control List | Lista de reglas de acceso en un router o firewall |
| VLAN | Virtual LAN | Red virtual separada sobre los mismos switches |
| CVE | Common Vulnerabilities and Exposures | Identificador público de una vulnerabilidad |
| IDS | Intrusion Detection System | Sistema de detección de intrusos |
| XML | eXtensible Markup Language | Formato de salida legible por programas |
| SOC | Security Operations Center | Centro de operaciones de seguridad |

## 2. Especificar objetivos

| Forma | Ejemplo |
|---|---|
| Rango | `192.168.0.1-10` |
| Subred | `192.168.0.1/24` (escanea la subred completa, desde la `.0`) |
| Nombre | `example.thm` |
| Solo listar, sin escanear | `nmap -sL 192.168.0.1/27` |

```bash
nmap -sL 192.168.0.1/27 | tail -3     # confirmar la última IP del rango
```

## 3. Subredes: lo aprendido en esta sala

Receta:
1. IPv4 = 32 bits. Bits libres = **32 − prefijo**.
2. Direcciones = **2^(bits libres)**. La base siempre es 2 (cada bit tiene 2 valores).
3. Máscara (último octeto) = **256 − direcciones**.
4. Los bloques parten en múltiplos del tamaño y **se cuentan desde 0**.
5. Equipos utilizables = direcciones − 2 (red y broadcast).

| CIDR | Direcciones | Máscara | Bloques en el octeto |
|---|---|---|---|
| /24 | 256 | 255.255.255.0 | 1 |
| /25 | 128 | 255.255.255.128 | 2 |
| /26 | 64 | 255.255.255.192 | 4 |
| /27 | 32 | 255.255.255.224 | 8 |
| /28 | 16 | 255.255.255.240 | 16 |
| /29 | 8 | 255.255.255.248 | 32 |
| /30 | 4 | 255.255.255.252 | 64 |
| /32 | 1 | 255.255.255.255 | Un solo host |

El inicio del último bloque coincide con la máscara (/28 → `.240`).

**Ejercicios resueltos:**

| Caso | Bloque | Red | Broadcast | Equipos |
|---|---|---|---|---|
| `.13` en /29 | .8 – .15 | .8 | .15 | 6 |
| `.50` en /28 | .48 – .63 | .48 | .63 | 14 |

**Errores corregidos:**
- Contar bloques de /29 como `0-8, 9-17` (9 direcciones). Correcto: `0-7, 8-15`.
- Calcular 4² en vez de 2⁴ para un /28 (y llegar a 32 direcciones). Correcto: 2⁴ = 16.
- Confundir la máscara del /28 con 255.255.255.255 (eso es /32).
- Creer que el broadcast es siempre `.255`: solo en /24. Cada bloque tiene su propia red y su propio broadcast.

**Clave:** dos equipos se ven directamente solo si están en el **mismo bloque**, no basta con tener la misma máscara. Con un /28, `.20` (bloque .16–.31) y `.40` (bloque .32–.47) necesitarían un router.

**Segmentación:** separar en bloques ordena la red, pero **un router rutea todo por defecto**. La protección real viene de las reglas (ACL o firewall) entre segmentos: por ejemplo, usuarios solo al puerto 445 del servidor de archivos, invitados solo a internet y usuarios entre sí bloqueados (frena el movimiento lateral).

## 4. Descubrimiento de hosts (`-sn`)

| Objetivo | Método |
|---|---|
| Red local | **ARP**: el firewall de Windows no lo bloquea; muestra también la MAC y el fabricante (OUI) |
| Red remota (detrás de un router) | ICMP echo, ICMP timestamp, TCP SYN al 443, TCP ACK al 80 |

Error propio: un `nmap -sn` contra la red del ejemplo de la sala quedó esperando; esa red no existía en el entorno.

## 5. Escaneo de puertos

| Opción | Tipo | Detalle |
|---|---|---|
| `-sT` | Connect | Handshake completo; sin root; más lento y **registrado en logs** |
| `-sS` | SYN ("sigiloso") | SYN → SYN-ACK → RST; requiere root; por defecto con `sudo` |
| `-sU` | UDP | Puertos UDP |
| `-F` | Rápido | 100 puertos más comunes |
| `-p 8008`, `-p 1-1000`, `-p-` | Rango | Puerto, rango o todos (65.535) |
| `-Pn` | Sin descubrimiento | Escanea aunque el host no responda al ping |

Sin `sudo`, Nmap no puede armar paquetes crudos y cae al escaneo por conexión completa.

**Observado en el objetivo:** además de SSH y un servidor web en un puerto no estándar, estaban abiertos servicios muy antiguos (echo 7, discard 9, daytime 13, qotd 17). En una auditoría real serían hallazgos.

## 6. Detección de servicios y sistema operativo

| Opción | Qué hace |
|---|---|
| `-sV` | Lee el banner de cada puerto abierto: software y versión |
| `-O` | Estima el sistema operativo (es una aproximación) |
| `-A` | `-O` + `-sV` + scripts + traceroute |

La columna `SERVICE` sin `-sV` es una **suposición** por número de puerto; `-sV` le pregunta al servicio. Es lo mismo que leer banners a mano con telnet.

```bash
nmap -sV -n -p 8008 <IP>     # solo el puerto web: ~6 s frente a ~158 s del escaneo completo
```

**Error propio:** se escribió mal el nombre del servidor detectado (dos letras invertidas). Regla: **copiar y pegar datos técnicos**, nunca tipearlos. Es el mismo mecanismo que explotan los dominios de phishing parecidos.

## 7. Velocidad (timing)

| Plantilla | Nombre | Tiempo para 100 puertos (ejemplo de la sala) |
|---|---|---|
| `-T0` | paranoid | ~9,8 h (5 min entre puertos) |
| `-T1` | sneaky | ~27,5 min |
| `-T2` | polite | ~40,6 s |
| `-T3` | normal (por defecto) | ~0,15 s |
| `-T4` | aggressive | ~0,13 s |
| `-T5` | insane | — |

Por nombre se escribe `-T <nombre en inglés>`. Error propio: escribir el comando completo y el número equivocado en vez de solo la opción pedida.

Otras: `--min-rate` / `--max-rate` (paquetes por segundo), `--min-parallelism` / `--max-parallelism`, `--host-timeout`.

## 8. Salida y reportes

| Opción | Qué hace |
|---|---|
| `-v`, `-vv` | Más detalle (verbose); tecla `v` durante el escaneo |
| `-d` ... `-d9` | Depuración (debugging) |
| `-oN` / `-oX` / `-oG` | Salida normal / XML / grepable |
| `-oA nombre` | Los tres formatos a la vez |

Patrón general: muchas opciones son la primera letra de la palabra en inglés (*verbose*, *debugging*, *output*).

## 9. Lecturas SOC

- **Reconocimiento:** una IP que toca muchos puertos de un servidor en poco tiempo es un escaneo; muchos RST de vuelta son puertos cerrados respondiendo.
- **Escaneos lentos (`-T0`/`-T1`)** quedan bajo los umbrales típicos de "X puertos en Y segundos". Que no haya alerta no significa que no hubo escaneo: se buscan patrones en ventanas largas.
- **`-sT` deja conexiones completas en los logs**; `-sS` deja SYN sin terminar, más visibles en firewall o captura.
- **Banners con versión** permiten buscar CVE exactos; ocultarlos es parte del hardening.
- **Guardar siempre con `-oA`:** un escaneo solo en pantalla no es evidencia; el archivo se adjunta al caso, se compara en el tiempo y se le calcula el hash.

## 10. Preguntas de repaso

1. ¿Qué método usa `-sn` en una red local y por qué funciona aunque el firewall bloquee el ping?
2. ¿Cuántas direcciones y equipos tiene un /27? ¿Cuál es su máscara?
3. ¿En qué bloque de un /28 cae la `.100`? Red, broadcast y equipos.
4. Dos equipos tienen la misma máscara /28 pero no se ven. ¿Por qué?
5. ¿Por qué segmentar con un router sin reglas protege poco?
6. Diferencia entre `-sS` y `-sT` en permisos y en huella.
7. ¿Por qué la columna SERVICE sin `-sV` no es confiable?
8. ¿Qué gana un atacante con `-T1` y cómo lo compensas como analista?
9. ¿Por qué guardar los escaneos con `-oA`?
10. ¿Qué hallazgo reportarías si ves los puertos 7, 9, 13 y 17 abiertos?

<details>
<summary><strong>Respuestas (abrir después de intentarlo)</strong></summary>

1. **ARP**. El firewall de Windows filtra tráfico IP (ICMP, TCP, UDP); ARP es capa 2 y el equipo tiene que responderlo para poder comunicarse.
2. 32 − 27 = 5 bits → 2⁵ = **32 direcciones**, **30 equipos**, máscara **255.255.255.224**.
3. Bloques de 16: .0, .16, .32, .48, .64, .80, **.96 – .111**. Red **.96**, broadcast **.111**, **14 equipos** (.97 a .110).
4. Están en bloques distintos. Ej.: `.20` (bloque .16–.31) y `.40` (bloque .32–.47): cada uno cree que el otro está "afuera" y manda el paquete al router.
5. Porque el router rutea todo por defecto: sin ACL o reglas de firewall, los segmentos se siguen hablando. Solo se gana orden y menos broadcast.
6. `-sS`: necesita root, no completa el handshake (SYN → SYN-ACK → RST), deja menos registro. `-sT`: sin root, handshake completo, queda en los logs del objetivo.
7. Es una suposición por número de puerto, no una consulta. Ej.: un SSH escuchando en el 443 aparecería como `https` sin `-sV`.
8. Pasa bajo los umbrales de detección ("X puertos en Y segundos"). Se compensa buscando patrones en ventanas largas: la misma IP tocando un puerto distinto cada 5 minutos durante toda la noche.
9. Porque un escaneo en pantalla no es evidencia. El archivo se adjunta al caso, se compara con escaneos posteriores (¿apareció un puerto nuevo?) y se le calcula el hash.
10. Servicios obsoletos e innecesarios (echo, discard, daytime, qotd): aumentan la superficie de ataque y algunos se pueden abusar para ataques de amplificación. Recomendación: deshabilitarlos.

</details>
