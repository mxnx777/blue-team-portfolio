# TryHackMe — Wireshark: The Basics

**Fecha:** 23 de noviembre de 2026 · **Ruta:** Cyber Security 101 → Wireshark: The Basics
**Entorno:** máquina de la sala con Wireshark; `http1.pcapng` (práctica) y `Exercise.pcapng` (preguntas)
**Propósito:** manejar la interfaz de Wireshark, leer un paquete capa por capa, extraer archivos y evidencia de una captura y usar filtros, con lectura SOC de lo encontrado.

> Sin flags ni respuestas literales de la sala. Los nombres de artistas, hashes y conteos que pide la sala se omiten; los ejemplos citados son del archivo de práctica o de la estructura de los campos.
> Base previa: serie de redes ([Core Protocols](networking-core-protocols-guia-soc.md), [Secure Protocols](networking-secure-protocols-guia-soc.md)).

## 1. Diccionario de siglas

| Sigla | Nombre en inglés | Significado |
|---|---|---|
| PCAP | Packet Capture | Archivo de captura de paquetes (`.pcap`, `.pcapng`) |
| IDS | Intrusion Detection System | Sistema de detección de intrusos |
| OSI | Open Systems Interconnection | Modelo de 7 capas |
| SHA256 / SHA1 | Secure Hash Algorithm | Funciones de hash (huella del archivo) |
| MD5 | Message Digest 5 | Función de hash antigua, aún usada para identificar archivos |
| TTL | Time To Live | Tiempo de vida del paquete IP |
| JPEG | Joint Photographic Experts Group | Formato de imagen |
| XML | eXtensible Markup Language | Lenguaje de marcado |
| ASCII | American Standard Code for Information Interchange | Codificación de texto |
| NTP | Network Time Protocol | Sincronización de relojes |
| LFI | Local File Inclusion | Inclusión de archivos locales |
| SOC | Security Operations Center | Centro de operaciones de seguridad |

## 2. Qué es y qué no es Wireshark

- Analizador de paquetes: captura tráfico en vivo y abre capturas guardadas.
- Usos: problemas de red (congestión, fallas), anomalías de seguridad (hosts no autorizados, puertos inusuales, tráfico sospechoso) y estudio de protocolos.
- **No es un IDS:** no alerta ni modifica paquetes, solo los muestra. Detectar la anomalía depende del analista.

## 3. Interfaz

| Zona | Función |
|---|---|
| Barra de herramientas | Captura, apertura, filtrado, exportación, combinación |
| Barra de filtro de visualización | Consultas para mostrar solo lo relevante |
| Archivos recientes | Reabrir capturas con doble clic |
| Filtro de captura e interfaces | Qué grabar y desde qué tarjeta (`eth0`, `lo`, `any`) |
| Barra de estado | Perfil, paquetes totales / mostrados, comentarios |

Paneles con una captura abierta:

| Panel | Contenido |
|---|---|
| Lista de paquetes | Resumen por paquete: número, tiempo, origen, destino, protocolo, info |
| Detalles | El paquete desglosado por capas |
| Bytes | Hexadecimal y ASCII; resalta los bytes del campo seleccionado |

**Colores:** reglas permanentes (*View → Coloring Rules*) y temporales (*Colourise Conversation*). Ejemplo: **Bad TCP** en negro con letras rojas marca retransmisiones, ACK duplicados y segmentos con problemas.

Otras funciones: captura en vivo (tiburón azul inicia, rojo detiene, verde reinicia) y combinación de capturas (*File → Merge*).

## 4. Propiedades del archivo de captura

*Statistics → Capture File Properties*, o el ícono del extremo inferior izquierdo.

| Sección | Información |
|---|---|
| File | Nombre, tamaño, **hashes** (SHA256, RIPEMD160, SHA1) |
| Time | Primer y último paquete, tiempo transcurrido |
| Capture | Sistema y aplicación que capturó |
| Interfaces | Interfaz, tipo de enlace, límite de tamaño |
| Statistics | Paquetes capturados y mostrados, bytes, promedios |
| Comments | Comentarios del archivo y de paquetes |

Hash por terminal (debe coincidir con el de Wireshark):

```bash
sha256sum Exercise.pcapng
```

**Lo observado:**
- El comentario de un paquete repetía un texto señuelo; la instrucción real estaba al final del comentario. Leer completo antes de concluir.
- La captura abarcaba **más de 18 años** entre primer y último paquete: casi con seguridad son capturas combinadas o relojes desfasados.

**Lectura SOC:**
- El **SHA256** de una captura se registra al obtenerla: es parte de la cadena de custodia. Un byte modificado cambia el hash completo. SHA1 ya no se recomienda para evidencia (colisiones conocidas).
- Las marcas de tiempo son la base de la línea de tiempo de un incidente. Fechas incoherentes obligan a revisar el origen de los datos antes de correlacionar; en servidores se usa NTP.
- Se analiza una **copia**, nunca el original, y se verifica con el hash.

## 5. Disección de un paquete por capas

Ir a un paquete: *Go → Go to Packet* (`Ctrl+G`). Abrir todas las capas: clic derecho → *Expand All* (`Ctrl+Right`).

| Línea en Wireshark | Capa | Campos útiles |
|---|---|---|
| `Frame N` | 1, física | `Arrival Time`, longitud, protocolos en el paquete, regla de color |
| `Ethernet II` | 2, enlace | MAC de origen y destino |
| `Internet Protocol Version 4` | 3, red | IP de origen y destino, **`Time to Live`**, protocolo |
| `Transmission Control Protocol` | 4, transporte | Puertos, Seq/Ack, flags, **`TCP payload (N bytes)`** |
| `[N Reassembled TCP Segments]` | Reensamblado | Respuesta grande unida a partir de varios paquetes |
| `Hypertext Transfer Protocol` | Aplicación | Código de estado, `Server:`, `ETag:`, `Content-Length` |
| `Line-based text data` / `eXtensible Markup Language` | Datos | Cuerpo de la respuesta |

**Lo observado y errores corregidos:**
- **Formato de fecha:** la sala pedía mes/día/año y se respondió primero en formato chileno día/mes. Las herramientas y logs usan formato estadounidense o ISO (`2004-05-13`). Confundirlos desplaza eventos en la línea de tiempo.
- **TTL recibido de 47:** se redondea al valor inicial más cercano (64), lo que sugiere un emisor Linux a unos 17 saltos. La geolocalización IP del paquete (servidor en EE. UU., cliente en Alemania) es coherente con esa distancia.
- **Payload vs reensamblado:** se abrió primero la sección de 14 segmentos reensamblados (respuesta completa) en vez de la carga útil del paquete individual. Comprobación: 13 segmentos × 1380 bytes + el último pedazo = total reensamblado. El dato del paquete calza con el total.

## 6. Navegación, búsqueda y extracción

| Acción | Cómo | Para qué |
|---|---|---|
| Buscar texto | *Edit → Find Packet* (`Ctrl+F`) → *Packet details* + *String* | Encontrar un valor en los campos de los paquetes |
| Exportar bytes de un campo | Clic derecho en el campo → *Export Packet Bytes* (`Ctrl+Shift+X`) | Extraer un archivo de un paquete (p. ej. una imagen) |
| Exportar objetos | *File → Export Objects → HTTP* (filtro de texto `.txt`, etc.) | Recuperar todos los archivos transferidos por HTTP |
| Información experta | *Analyze → Expert Information* | Resumen de errores, advertencias y notas |

```bash
md5sum imagen-extraida.jpg     # huella del archivo extraído
cat archivo-recuperado.txt     # leer un archivo de texto recuperado
```

**Errores corregidos:**
- Al exportar la imagen se seleccionó primero un sub-segmento JPEG; hay que hacer clic derecho en la línea padre `JPEG File Interchange Format` (el panel de bytes debe resaltar el archivo completo). Exportar otra capa cambia el hash.
- Archivos guardados con espacios en el nombre: `ls` los muestra entre comillas y hay que citarlos en los comandos. Convención: nombres sin espacios.

**Niveles de Expert Information:**

| Nivel | Color | Ejemplo |
|---|---|---|
| Error | Rojo | Paquetes mal formados |
| Warning | Amarillo | Comportamiento inusual |
| Note | Celeste | Retransmisiones |
| Chat | Azul | Inicio o cierre de conexiones |

**Lectura SOC de lo observado:**
- La captura incluía tráfico hacia `testphp.vulnweb.com`, un sitio deliberadamente vulnerable. Aparecieron dos patrones de ataque web en las URL: un parámetro numérico (`artists.php?artist=1`), objetivo típico de **SQL injection**, y un parámetro con ruta de archivo (`showimage.php?file=./pictures/...`), objetivo típico de **path traversal / LFI** (en logs se reconoce por `../` repetidos).
- La gran mayoría de las advertencias eran caracteres ilegales en nombres de cabeceras HTTP. **Hipótesis (no verificada):** tráfico de una herramienta de escaneo web que envía pedidos deformados. Expert Information indica dónde mirar; la conclusión requiere revisar esos paquetes.
- *Export Objects* es como se recupera desde una captura un malware descargado por un equipo infectado, para luego calcular su hash y consultar su reputación.
- Nombres de archivo con espacios o sin extensión en directorios como `/tmp` son una técnica de ocultamiento; a Linux no le importa la extensión, sino el contenido.

## 7. Filtrado de paquetes

| Tipo | Qué hace |
|---|---|
| Filtro de **captura** | Decide qué se graba; lo excluido no queda en el archivo |
| Filtro de **visualización** | Oculta lo que no interesa de lo ya grabado; se quita con la X |

**Regla de oro:** *si puedes hacerle clic, puedes filtrarlo*.

| Opción (clic derecho) | Efecto |
|---|---|
| Apply as Filter | Filtra de inmediato por el valor seleccionado |
| Prepare as Filter | Escribe el filtro y espera Enter (para combinar con *and/or*) |
| Conversation Filter | Solo la conversación entre esas IP y puertos |
| Colourise Conversation | Pinta la conversación sin ocultar el resto |
| Apply as Column | Agrega el campo como columna en la lista |
| Follow Stream | Reconstruye la conversación (rojo = cliente, azul = servidor) y aplica un filtro `tcp.stream eq N` |

Filtros básicos:

```text
http                     # por protocolo
tcp.port == 80           # por puerto TCP
udp.port == 53           # por puerto UDP
ip.addr == 192.168.1.2   # por IP (origen o destino)
```

**Lo observado:**
- Aplicar el protocolo HTTP como filtro desde los detalles de un paquete redujo la vista de 58.620 a una pequeña fracción de los paquetes (barra de estado: `Displayed`).
- *Follow HTTP Stream* sobre una respuesta del sitio mostró el listado de artistas en el HTML. Trampa: el mismo artista aparece varias veces; hay que contar valores **distintos**. El cuadro *Find* de la ventana del flujo ayuda a recorrer las coincidencias.

## 8. Lo que hay que saber sí o sí

- Wireshark no es un IDS.
- Hash SHA256 de la captura = cadena de custodia; trabajar sobre copias.
- Orden de capas en el panel de detalles y dónde está cada campo (fecha en Frame, TTL en IP, payload en TCP, cabeceras en HTTP).
- Payload de un paquete ≠ total reensamblado.
- Extracción: *Export Packet Bytes* (un campo) vs *Export Objects* (todos los archivos HTTP).
- Filtro de captura vs filtro de visualización.
- Formatos de fecha en herramientas: mes/día o año-mes-día.

## 9. Preguntas de repaso

1. ¿Por qué Wireshark no es un IDS?
2. ¿Para qué sirve registrar el SHA256 de una captura y por qué no SHA1?
3. Una captura dura 18 años. ¿Qué hipótesis planteas antes de usar sus tiempos?
4. ¿En qué capa del panel de detalles encuentras el TTL? ¿Y la carga útil TCP?
5. Un TTL recibido es 116. ¿Qué sistema y cuántos saltos estimas?
6. ¿Qué diferencia hay entre la carga útil de un paquete y los segmentos reensamblados?
7. ¿Qué diferencia hay entre *Export Packet Bytes* y *Export Objects → HTTP*?
8. Ves `?file=../../../../etc/passwd` en una URL. ¿Qué ataque es?
9. ¿Qué indica una gran cantidad de advertencias del mismo tipo en Expert Information y qué haces después?
10. Explica la diferencia entre filtro de captura y de visualización.
11. ¿Qué hace *Prepare as Filter* que no hace *Apply as Filter*?
12. En *Follow Stream*, ¿qué color corresponde al cliente y cuál al servidor?

## 10. Pendientes

- Sala completada (la task de conclusión no tiene contenido evaluado).
- Siguiente sala: *Wireshark: Packet Operations* (filtros avanzados, estadísticas).
