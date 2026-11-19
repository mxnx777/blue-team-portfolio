# ICMP, TTL y ARP: qué ve el firewall de Windows y qué no

**Fecha:** 2026-11-19 · **Entorno:** BLUELAB (red aislada `172.31.250.0/24`)
**Origen:** repaso práctico de los protocolos vistos en TryHackMe (DHCP, ARP, ICMP, enrutamiento, NAT)

## Objetivo

Comprobar en el laboratorio, con capturas de tráfico, tres cosas:

1. Cómo se ve un `ping` (ICMP) en el cable y qué información entrega.
2. Qué revela el **TTL** de una respuesta sobre el sistema operativo remoto.
3. Por qué el firewall de Windows bloquea el ping pero **no impide descubrir el equipo** dentro de la misma red (ARP).

## Topología usada

| Equipo | IP | MAC | Notas |
|---|---|---|---|
| Kali Linux | `172.31.250.40/24` (`eth0`) | `00:0c:29:1c:e8:30` | `eth1` (NAT de VMware) **desconectada** durante el ejercicio |
| LAB-CLIENT01 (Win 11) | `172.31.250.20/24` | `00:0c:29:38:5f:64` | Sin puerta de enlace: red sin salida |

VMs encendidas: solo Kali y LAB-CLIENT01 (DC01 y WAZUH-SRV01 apagados).

## 0. Verificación previa del aislamiento

Kali tiene dos adaptadores: `LAB_ISOLADO` + NAT. Un host con dos redes (*dual-homed*) puede convertirse en puente entre el lab e Internet, así que se verificó que no estuviera ruteando:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

Resultado: `0`. Kali no reenvía paquetes entre interfaces.

```bash
ip -br a
```

| Interfaz | Estado | Dirección |
|---|---|---|
| `lo` | UNKNOWN | `127.0.0.1/8` |
| `eth0` | UP | `172.31.250.40/24` |
| `eth1` | DOWN | — |

Decisión: el adaptador NAT se deja con **"Conectar al encender" desmarcado** y se conecta solo para instalar o actualizar.

## 1. ICMP: el ping visto en el cable

**Sniffer en Kali:**

```bash
sudo tcpdump -i eth0 -n icmp
```

- `-i eth0`: escucha solo en la interfaz del lab.
- `-n`: no resuelve nombres (sin DNS, sin tráfico extra).
- `icmp`: filtro; muestra solo ICMP.

**Ping desde LAB-CLIENT01 hacia Kali** (en este sentido, porque el firewall de Windows bloquea el ping entrante por defecto):

```cmd
ping -n 4 172.31.250.40
```

**Captura (extracto):**

```text
07:34:43.156416 IP 172.31.250.20 > 172.31.250.40: ICMP echo request, id 1, seq 1, length 40
07:34:43.156437 IP 172.31.250.40 > 172.31.250.20: ICMP echo reply, id 1, seq 1, length 40
```

| Campo | Significado |
|---|---|
| `07:34:43.156416` | Marca de tiempo con microsegundos; base de una línea de tiempo |
| `.20 > .40` | Origen > destino |
| `echo request` / `echo reply` | ICMP tipo 8 / tipo 0 |
| `id 1` | Identificador del proceso `ping`; igual en los 4 paquetes |
| `seq 1..4` | Secuencia; empareja cada respuesta con su pregunta |
| `length 40` | 32 bytes de datos (Windows) + 8 de cabecera ICMP |

**Observaciones:**

- Respuesta en ~21 µs (red virtual local).
- Ritmo de 1 request por segundo: comportamiento normal de `ping`. Un barrido (*ping sweep*) dispara muchos requests por segundo hacia muchas IPs; esa diferencia de ritmo es lo que busca una regla de detección.
- Linux manda 56 bytes de datos (`56(84) bytes`: 56 + 8 ICMP + 20 IP) y Windows 32. El tamaño también deja huella de qué sistema originó el ping.

## 2. TTL: huella pasiva del sistema operativo

En el ping de Windows → Kali, las respuestas llegaron con **`TTL=64`**:

```text
Respuesta desde 172.31.250.40: bytes=32 tiempo=1ms TTL=64
```

La respuesta la emite Kali. TTL inicial por defecto:

| Sistema | TTL inicial |
|---|---|
| Linux | 64 |
| Windows | 128 |
| Equipos de red (Cisco, etc.) | 255 |

Cada router resta 1. Aquí no hay router, por eso llega intacto.

**Prueba inversa (Kali → Windows):**

```bash
ping -c 4 172.31.250.20
```

Primer intento: `100% packet loss`. El equipo estaba encendido; el firewall de Windows **descarta en silencio** el echo request entrante.

> Error propio documentado: `ping -c 172.31.250.20` devuelve `invalid argument`. `-c` (*count*) exige un número a continuación: `ping -c 4 <IP>`.

Para comprobar el TTL se abrió temporalmente **solo** ICMP tipo 8 entrante (CMD como administrador):

```cmd
netsh advfirewall firewall add rule name="LAB-ICMPv4-In" protocol=icmpv4:8,any dir=in action=allow
```

- `protocol=icmpv4:8,any`: solo echo request, cualquier código; no abre nada más.
- `dir=in action=allow`: permite el tráfico entrante.

Con la regla activa, el Windows respondió con TTL 128, como se esperaba (observado en pantalla; no se guardó captura de esa salida). La regla se eliminó al terminar:

```cmd
netsh advfirewall firewall delete rule name="LAB-ICMPv4-In"
```

```text
Se eliminaron 1 reglas.
```

**Límites del método:**

- Si el firewall bloquea, **no hay respuesta y por lo tanto no hay TTL** que leer. Hay que buscar la huella en otro tráfico (p. ej. una respuesta TCP de un puerto abierto) o con `nmap -O`.
- Con routers de por medio el TTL llega descontado: se redondea hacia arriba al valor por defecto más cercano (120 → 128, Windows; 57 → 64, Linux).
- El TTL se puede modificar. Es una **pista**, no una prueba.

## 3. ARP: lo que el firewall de Windows no bloquea

Con la regla ICMP ya eliminada (ping bloqueado otra vez):

**Terminal 1 en Kali:**

```bash
sudo tcpdump -i eth0 -n -e arp
```

- `-e`: muestra direcciones MAC (capa 2).

**Terminal 2 en Kali:**

```bash
sudo ip neigh flush dev eth0     # vacía la tabla ARP de Kali
ping -c 2 172.31.250.20          # falla: ICMP bloqueado
```

```text
2 packets transmitted, 0 received, 100% packet loss
```

**Captura clave:**

```text
07:51:05.013388 00:0c:29:1c:e8:30 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 172.31.250.20 tell 172.31.250.40, length 28
07:51:05.013982 00:0c:29:38:5f:64 > 00:0c:29:1c:e8:30, ethertype ARP (0x0806), length 60: Reply 172.31.250.20 is-at 00:0c:29:38:5f:64, length 46
```

| Parte | Significado |
|---|---|
| `> ff:ff:ff:ff:ff:ff` | Broadcast: el request llega a todos los equipos del segmento |
| `who-has .20 tell .40` | "¿Quién tiene la .20? Avísale a la .40" |
| `00:0c:29:38:5f:64 > 00:0c:29:1c:e8:30` | El Windows responde en unicast, directo a Kali |
| `is-at 00:0c:29:38:5f:64` | "La .20 soy yo y esta es mi MAC" |

**Conclusión:** el ping falló, pero el Windows **igual reveló que existe** al responder ARP. El firewall de Windows filtra tráfico IP (ICMP/TCP/UDP); ARP opera en capa 2, por debajo de IP, y el host debe responderlo para poder comunicarse. Dentro de la misma red local, bloquear el ping no oculta el equipo. Herramientas como `nmap -sn` o `arp-scan` usan precisamente ARP en redes locales.

**Dato adicional:** el prefijo `00:0c:29` es el OUI de VMware. La MAC por sí sola indica que el equipo es una máquina virtual.

## 4. Hallazgo no planificado: el agente Wazuh buscando a su manager

En la misma captura, LAB-CLIENT01 emitía de forma continua, cada 1–2 segundos y **sin respuesta**:

```text
00:0c:29:38:5f:64 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 172.31.250.30 tell 172.31.250.20, length 46
```

Hipótesis: el agente Wazuh intenta contactar a su manager, que estaba apagado. Verificación en el cliente:

```cmd
findstr /i "address" "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

```text
<address>172.31.250.30</address>
```

**Confirmado:** el agente apunta a `172.31.250.30` (WAZUH-SRV01, apagado durante el ejercicio).

**Lectura SOC:** un ARP repetido sin respuesta hacia la misma IP es la huella en red de un equipo que intenta alcanzar un destino caído. En un entorno real puede indicar un agente que perdió a su servidor (pérdida de visibilidad del SIEM), un servicio dependiente caído o una configuración que apunta a una IP obsoleta.

## Qué quedó confirmado y qué no

| Punto | Estado |
|---|---|
| ICMP echo request/reply capturado y desglosado | Confirmado (captura) |
| TTL 64 desde Kali | Confirmado (captura) |
| TTL 128 desde Windows con regla temporal | Observado, sin captura guardada |
| Firewall de Windows descarta el ping entrante sin responder | Confirmado |
| ARP responde aunque el ICMP esté bloqueado | Confirmado (captura) |
| Agente Wazuh apunta a `172.31.250.30` | Confirmado (`ossec.conf`) |
| Estado de la entrada ARP en `ip neigh show dev eth0` | **Pendiente**: no se ejecutó |
| Cómo obtiene IP LAB-CLIENT01 (DHCP o fija) | **Pendiente**: revisar con `ipconfig /all` |

## Preguntas de repaso

1. ¿Por qué un `100% packet loss` no prueba que un equipo esté apagado?
2. Un host responde con TTL 119. ¿Qué sistema operativo es más probable y cuántos saltos hay?
3. ¿En qué capa opera ARP y por qué el firewall de Windows no lo filtra?
4. ¿Qué diferencia hay entre el destino de un ARP request y el de un ARP reply?
5. ¿Qué te dice el prefijo `00:0c:29` en una MAC?
6. Ves en una captura ARP repetidos hacia la misma IP sin respuesta. ¿Qué hipótesis planteas y cómo la verificas?
7. ¿Por qué un host con dos redes (*dual-homed*) es un riesgo, y cómo compruebas en Linux que no esté ruteando?

## Próximos pasos

- Ejecutar `ip neigh show dev eth0` y documentar el estado de la entrada (`REACHABLE` / `STALE`).
- Repetir el ejercicio con WAZUH-SRV01 encendido: medir si el ARP repetido desaparece y ver qué registra Wazuh de un `nmap -sn 172.31.250.0/24`.
- DHCP: identificar quién reparte IPs en `LAB_ISOLADO` (DHCP de VMware, DC01 o IPs fijas) y capturar el intercambio DORA.
