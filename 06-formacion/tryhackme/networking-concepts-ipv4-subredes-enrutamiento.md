# TryHackMe — Networking Concepts, Task 4: IPv4, subredes y enrutamiento

**Fecha:** 16 de noviembre de 2026 · **Ruta:** Cyber Security 101 → Networking Concepts → Task 4 «IP Addresses and Subnets»  
**Estado de la sala comprobado en capturas:** dos respuestas de esta Task marcadas **correctas**.  
**Propósito:** estudiar después los fundamentos de IPv4, las máscaras, el prefijo CIDR, las IP privadas y el enrutamiento; conectar conceptos con análisis Blue Team / SOC N1.

> Los ejemplos `192.168.66.89`, las salidas `ifconfig`/`ip a s` y el diagrama de routers proceden del material visto en TryHackMe. Los apartados de ejercicios adicionales son **propuestas didácticas**, no pruebas ejecutadas por el alumno en la sala.

## 1. Diccionario de siglas: leer varias veces

| Sigla | Nombre en inglés | Significado o aplicación |
|---|---|---|
| IP | Internet Protocol | Protocolo de Internet; direccionamiento y transmisión de paquetes |
| IPv4 | Internet Protocol version 4 | IP versión 4: dirección de 32 bits |
| IPv6 | Internet Protocol version 6 | IP versión 6: dirección de 128 bits |
| TCP | Transmission Control Protocol | Transporte con conexión; OSI capa 4 |
| CIDR | Classless Inter-Domain Routing | Enrutamiento entre dominios sin clases; prefijos como `/24` |
| NAT | Network Address Translation | Traducción de direcciones de red |
| DHCP | Dynamic Host Configuration Protocol | Asignación dinámica de parámetros IP |
| MAC | Media Access Control | Dirección de enlace; OSI capa 2 |
| LAN | Local Area Network | Red de área local |
| WAN | Wide Area Network | Red de área amplia |
| RFC | Request for Comments | Documento técnico de protocolos de Internet |
| SOC | Security Operations Center | Centro de Operaciones de Seguridad |

## 2. IPv4: cuatro octetos, 32 bits

IPv4 emplea 4 números separados por puntos, por ejemplo:

```text
192 . 168 . 66 . 89
 8     8    8    8  bits
```

- Cada número es un **octeto**, porque contiene **8 bits**.
- Cada octeto puede valer de **0 a 255**, inclusive.
- Cuatro octetos = **32 bits**.
- Cantidad teórica total de direcciones IPv4: **2^32 = 4.294.967.296**. Esto incluye direcciones reservadas y no todas son utilizables como hosts públicos.

**Nota de precisión:** no es correcto afirmar que «0 y 255 siempre son direcciones de red y difusión» para cualquier IP. Eso depende de **la máscara de subred**. En la red **192.168.66.0/24**, efectivamente **192.168.66.0** es red y **192.168.66.255** es broadcast. En otras máscaras la situación cambia. Asimismo, **192.168.305.19 es inválida** porque `305 > 255` en un octeto.

### Direcciones IPv4 vistas en el material

- `192.168.0.1` y `172.16.159.243`: ambas sintácticamente válidas.
- `192.168.1.1`: ejemplo de una IP de host en una red doméstica, según máscara.
- `192.168.66.89`: dirección del host en la captura de TryHackMe.
- `192.168.305.19`: inválida: tercer octeto supera 255.

**Regla para memorizar:** *Cuatro números; cada uno entre cero y doscientos cincuenta y cinco. La máscara determina cuál es la red.*

## 3. ¿Qué es una subred y por qué sirve?

Una **subred** divide un espacio de direcciones IP en redes lógicas de menor tamaño. La **máscara de subred** identifica qué bits corresponden al prefijo de red y cuáles son bits de host.

En la sala se utiliza:

```text
IP:       192.168.66.89
Máscara:  255.255.255.0
Prefijo:  /24
```

**/24 significa 24 bits de red**, no 24 equipos; la IP completa tiene 32 bits, quedan **8 bits de host**.

### Cálculo paso a paso del ejemplo /24

1. Red fija: `192.168.66` (tres octetos de 8 bits = 24 bits).
2. Octeto final variable: de `0` a `255`.
3. **Dirección de red:** `192.168.66.0`.
4. **Broadcast (difusión):** `192.168.66.255`.
5. **Rango tradicional de hosts utilizables:** `192.168.66.1` hasta `192.168.66.254`.
6. **Total teórico:** `2^8 = 256` direcciones dentro de esa subred.
7. **Hosts típicos utilizables:** `256 - 2 = 254`, excluyendo red y difusión en este ejemplo IPv4 /24.

Este rango corresponde al **ejemplo concreto /24**. Para prefijos como /30, /31 y /32 se aplican detalles especiales; no extrapolar ciegamente la fórmula sin revisar el caso.

### ¿Están en la misma red?

Con máscara `255.255.255.0` (/24):

| Equipo | IP | Red calculada |
|---|---|---|
| A | `192.168.66.89` | `192.168.66.0/24` |
| B | `192.168.66.120` | `192.168.66.0/24` |
| C | `192.168.67.8` | `192.168.67.0/24` |

A y B comparten subred; C no. Esto **no garantiza** comunicación efectiva: faltan condiciones como enlace, VLAN, firewall y configuración.

## 4. Consultar configuración en Windows y Linux

### Windows — comandos ya estudiados en la sala Windows Command Line

```cmd
ipconfig
ipconfig /all
route print
```

- **`ipconfig`:** muestra IPv4, máscara y puerta de enlace, entre otros datos.
- **`ipconfig /all`:** añade MAC (Physical Address), DNS, DHCP, interfaces y otros detalles.
- **`route print`:** tabla de enrutamiento de Windows: ayuda a conocer por qué interfaz/siguiente salto se envía tráfico.

### Linux — ejemplos de la Task 4

TryHackMe presentó tanto `ifconfig` como `ip a s` (forma abreviada de `ip address show`):

```bash
ifconfig
ip address show
ip a s
```

Salida de la sala (fragmento relevante de `ifconfig`):

```text
inet 192.168.66.89  netmask 255.255.255.0  broadcast 192.168.66.255
```

Salida de la sala (fragmento relevante de `ip a s`):

```text
inet 192.168.66.89/24 brd 192.168.66.255 scope global dynamic ... wlo1
```

**Interpretación de campos:**

| Campo | Interpretación |
|---|---|
| `inet` | Dirección IPv4 asignada |
| `192.168.66.89/24` | IP más prefijo CIDR |
| `netmask` | Máscara IPv4 |
| `brd` o `broadcast` | Dirección de difusión |
| `dynamic` | Dirección asignada dinámicamente |
| `wlo1` | Nombre de interfaz inalámbrica mostrado en la sala |
| `link/ether` | Dirección MAC de la interfaz |
| `inet6` | Dirección IPv6 |
| `RX` y `TX` | Contadores de paquetes recibidos y transmitidos |

**Importante:** `ifconfig` no está instalado en todas las distribuciones actuales; `ip` suele ser la herramienta moderna recomendada.

### Mini ejercicio propuesto en .BLUELAB

En Windows 11 de laboratorio (no necesita alterar configuración):

```powershell
ipconfig /all
route print
Get-NetIPConfiguration
```

Anota: interfaz usada, dirección IPv4, máscara/prefijo, gateway, DNS y si la asignación es dinámica. Antes de hacer cambios, compara los datos con la configuración del controlador de dominio y la red virtual VMware.

## 5. IP privada vs pública — RFC 1918

TryHackMe enseñó los tres **rangos reservados para uso privado** por RFC 1918:

| Espacio privado | Desde | Hasta |
|---|---|---|
| `10.0.0.0/8` | `10.0.0.0` | `10.255.255.255` |
| `172.16.0.0/12` | `172.16.0.0` | `172.31.255.255` |
| `192.168.0.0/16` | `192.168.0.0` | `192.168.255.255` |

**Ejercicio mental:** ¿`172.23.182.251` es privada? **Sí**, porque 23 cae entre 16 y 31 en el segundo octeto.  
**¿`172.40.0.1` es privada?** No pertenece al rango privado 172.16/12.  
**¿Toda IP fuera de estos rangos es pública enrutable?** No: existen otras reservas y rangos especiales, como loopback y enlace local.

### Ejercicio real de la Task 4: IP que NO es privada

Opciones mostradas en TryHackMe:
- `192.168.250.125`: privada (192.168/16).
- `10.20.141.132`: privada (10/8).
- `49.69.147.197`: **no pertenece a un rango privado RFC1918** ✅.
- `172.23.182.251`: privada (172.16/12).

**Respuesta aceptada:** `49.69.147.197`.

### ¿Qué es NAT?

**NAT = Network Address Translation (Traducción de Direcciones de Red).** Permite que un equipo intermedio, normalmente un router, traduzca direcciones IP y, en NAT de puertos, también puertos, cuando atraviesa el límite entre redes.

Ejemplo conceptual: ordenador `192.168.1.10` sale a Internet por un router que utiliza otra IP en su interfaz WAN. La dirección privada por sí sola normalmente **no se enruta en Internet pública**. No concluir que todas las IP privadas son inaccesibles desde cualquier otra red: entre redes privadas **sí puede haber rutas, VPN y autorizaciones**.

## 6. Enrutamiento — router y la capa 3 OSI

**Router o enrutador:** un equipo/dispositivo que evalúa direcciones IP y su tabla de rutas para decidir el **siguiente salto** (*next hop*) para reenviar paquetes entre redes.

La Task 4 mostró varios routers conectando **Office User**, **Mobile User** y **Web Server** por múltiples trayectos posibles.

Idea clave:

- Si la IP destino está **dentro de la subred local**, normalmente se utiliza entrega en el enlace (por ejemplo Ethernet).
- Si está **fuera de la subred**, el equipo normalmente envía el tráfico hacia una **puerta de enlace predeterminada** (*default gateway*) u otra ruta específica.
- Puede haber múltiples rutas; no siempre gana la que visualmente parece más corta. Se decide según tabla de rutas, prefijos y métricas/protocolos de enrutamiento.

### Ejemplo para practicar: diagnosticar una ruta

Windows:

```cmd
ipconfig /all
route print
tracert 1.1.1.1
```

Linux:

```bash
ip address show
ip route
traceroute 1.1.1.1
```

**Cuidado al interpretar `tracert`/`traceroute`:** un router puede no responder a las sondas sin que el tráfico final necesariamente falle. No deducir automáticamente un corte por cada asterisco.

## 7. Caso SOC N1 — separar problemas por capas

**Situación didáctica:** desde un equipo Windows queremos conectar con una IP de servidor `10.147.17.205` y puerto SQL `1433`.

1. **Capa 3 OSI / Internet TCP-IP:** revisar IP local, máscara, puerta de enlace y rutas.
2. **Capa 4 OSI / Transporte TCP-IP:** `Test-NetConnection 10.147.17.205 -Port 1433` intenta conexión TCP al puerto.
3. **Aplicación:** si TCP funciona, revisar autenticación SQL, servicio e instancia.
4. **Evidencia:** registrar hora, origen, destino, puerto y resultado exacto.
5. **Límite:** si falla TCP, todavía no sabemos si se debe a firewall, ruta, servicio o puerto equivocado.

**Hilo conductor:** la máscara determina qué es red local; el router decide la ruta; TCP usa el puerto; el servicio de aplicación interpreta la solicitud. Son problemas distintos y hay que probarlos por separado.

## 8. Preguntas para memorizar (sin mirar la clave)

1. ¿Qué significa IPv4 y cuántos bits tiene?
2. ¿Qué es un octeto? ¿Cuál es su rango decimal?
3. ¿Por qué `192.168.305.19` no es una IPv4 válida?
4. ¿Qué equivale a `255.255.255.0` en CIDR?
5. ¿Qué significa `/24`?
6. ¿Cuál es la dirección de red de `192.168.66.89/24`?
7. ¿Cuál es su broadcast?
8. ¿Qué rango de hosts utilizables tiene ese /24?
9. ¿Por qué no es verdad que los octetos 0 y 255 estén prohibidos en cualquier IP?
10. ¿Cuáles son los tres rangos privados RFC1918?
11. ¿`172.23.182.251` es privada?
12. ¿`49.69.147.197` es RFC1918?
13. ¿Qué hace `ipconfig /all`?
14. ¿Qué enseña `ip a s`?
15. ¿Qué significa `NAT`?
16. ¿En qué capa OSI trabaja típicamente un router IP?
17. ¿Qué es un gateway?
18. ¿Una IP privada puede hablar con otra red privada?
19. ¿Qué hace `tracert`?
20. ¿Qué NO demuestra un `TcpTestSucceeded = False`?

### Clave de autocorrección

1. Internet Protocol version 4; 32 bits. 2. Grupo de 8 bits; 0–255. 3. 305 excede 255. 4. /24. 5. 24 bits de prefijo/red. 6. 192.168.66.0. 7. 192.168.66.255. 8. 192.168.66.1–192.168.66.254. 9. El significado de red/broadcast depende de máscara. 10. 10/8; 172.16/12; 192.168/16. 11. Sí. 12. No. 13. Configuración detallada, IPv4, MAC, DNS, DHCP, etc. 14. Interfaces y direcciones (incluidos prefijos). 15. Network Address Translation. 16. Capa 3. 17. Siguiente salto por defecto hacia otras redes. 18. Sí, si existen rutas y autorización. 19. Sondea saltos de ruta. 20. No prueba por sí solo que servidor esté apagado ni que la contraseña SQL esté incorrecta.

## 9. Evidencia de la sala y próximos pasos

**Correctas en captura:**

| Pregunta | Respuesta | Evidencia |
|---|---|---|
| Which IP is not private? | `49.69.147.197` | TryHackMe: «Respuesta correcta» ✅ |
| Which IPv4 is invalid? | `192.168.305.19` | TryHackMe: «Respuesta correcta» ✅ |

**Próxima práctica útil:** distinguir `/24`, `/25` y `/26` con cálculos manuales; después ejecutar `ipconfig /all` y `route print` en .BLUELAB y documentar salida real sin datos privados de producción.

**Recordatorio repetible:** IPv4 = cuatro octetos = 32 bits. Máscara = frontera red/host. Router = capa 3. TCP = capa 4. TCP/IP y OSI organizan funciones relacionadas, pero no son lo mismo.
