# Apuntes de formación — Reconocimiento web, HTTP, logs y phishing
**Fecha:** 2026-11-03  
**Tipo:** teoría aplicada y definiciones — separado de `laboratorio/`  
**Objetivo:** poder volver a leer, entender el porqué y reconocer situaciones similares como analista SOC N1.

## 1. Mapa mental

**Seguridad ofensiva / Red Team:** simula técnicas de ataque de forma autorizada para descubrir debilidades.  
**Seguridad defensiva / Blue Team:** previene, monitorea, detecta, investiga y responde a incidentes.  
**SOC:** Security Operations Center, centro/equipo de operaciones de seguridad. **SOC N1:** triaje inicial, verificación, contexto, recopilación de evidencias y escalamiento.

Ruta de aprendizaje: **hipótesis → prueba controlada → evidencia → interpretación → respuesta → documentación**. Una prueba se repite solo si responde una pregunta nueva o verifica una corrección.

## 2. Reconocimiento web: DIRB

**DIRB:** herramienta que prueba rutas web usando un diccionario de nombres frecuentes (enumeración de contenido). Puede encontrar rutas que no estén enlazadas desde el menú. **Encontrar una URL no demuestra una vulnerabilidad.**

Ejemplo ilustrativo en un entorno autorizado:

```bash
dirb http://web-lab.local /usr/share/dirb/wordlists/common.txt
```

**Directory listing (listado de directorios):** respuesta del servidor que enumera nombres de archivos cuando falta una página índice. Un `Index of /css/` confirma listado público, pero mostrar CSS públicos suele tener impacto bajo. La exposición de backups, configuraciones o información privada podría ser mucho más grave.

**Document root:** carpeta del servidor desde la que se publican recursos web; en cPanel suele ser `public_html`. **Inherit** en Índices significa que hereda la política superior; **No Indexing** evita mostrar el listado, pero **no impide** que un archivo público conocido sea descargado directamente.

## 3. HTTP, curl y métodos

**HTTP:** protocolo de solicitudes y respuestas entre cliente y servidor. **HTTPS:** HTTP protegido por TLS.  
**URL:** dirección de un recurso. **Ruta:** parte de la URL, p. ej. `/docs/manual.pdf`.  
**GET:** solicita una representación de un recurso. **HEAD:** solicita los encabezados. **POST:** envía datos a un punto de aplicación. **DELETE:** solicita eliminar el recurso; el servidor puede rechazarlo.  
**curl:** cliente para realizar solicitudes HTTP y observar encabezados/cuerpo de respuesta.

Códigos habituales:
| Código | Interpretación |
| --- | --- |
| 200 | Solicitud respondida correctamente; **no prueba acceso autorizado** ni existencia del recurso exacto |
| 301 | Redirección permanente |
| 401 | Se requiere autenticación válida |
| 403 | Acceso prohibido |
| 404 | Recurso no encontrado |
| 405 | Método HTTP no permitido sobre ese recurso |
| 500 | Error del servidor; por sí solo no demuestra explotación |

**Falso positivo de enumeración:** si múltiples rutas inexistentes devuelven `200`, el mismo HTML y los mismos bytes, puede existir una página comodín, reescritura, caché o configuración de errores. Comparar ruta válida, ruta inventada, tipo MIME, tamaño y contenido antes de afirmar que existe un archivo.

**Por qué el nombre exacto importa:** `prueba` y `prueba.txt` son rutas distintas; en Linux también importa la capitalización.

**Importante:** descargar un archivo público no otorga permiso para modificarlo o borrarlo. El acceso de lectura, escritura y eliminación deben comprobarse por separado, sin inferir permisos de uno a otro.

## 4. Caso práctico observado (resumen sin datos sensibles)

Se utilizó una copia temporal de un sitio institucional en un hosting Apache/cPanel para ejercicios autorizados. Una ruta `/css/` entregó un índice público. Un archivo desechable dentro de una carpeta de prueba devolvió `200` y su contenido **solo al solicitar su nombre exacto**. Algunas rutas distintas devolvían el HTML principal con tamaño repetido; esto obligó a revisar las suposiciones, no a concluir que existían todos esos archivos. Se solicitó `DELETE` sobre el archivo desechable sin credenciales, Apache respondió **405**, y se confirmó que el archivo seguía presente.

**Conclusión demostrada:** lectura anónima de recursos publicados; listado de un directorio; el método DELETE no consiguió eliminar el archivo en esa configuración. **No demostrado:** acceso a cPanel, modificación no autorizada, exfiltración de datos privados, ejecución de código o compromiso del servidor.

**Higiene:** no guardar copias reales del sitio ni respaldos con secretos en la raíz pública; retirar material temporal tras conservar evidencia; no publicar IP públicas, tokens, claves API, logs completos ni capturas sensibles en un portafolio.

## 5. Logs de Apache y lectura SOC

Un registro de acceso permite reconstruir qué **solicitud** recibió el servidor y cómo respondió.

Ejemplo **ficticio y anonimizado**:
```text
192.0.2.44 - - [03/Nov/2026:11:07:56 -0300] "DELETE /ensayo/archivo.txt HTTP/1.1" 405 21871 "-" "curl/8.x"
```

Campos:
- **IP de origen:** dirección aparente del cliente; puede pertenecer a NAT, proxy o VPN, por lo que **no identifica a una persona**.
- **Timestamp / -0300:** fecha, hora y desplazamiento horario.
- **Método y ruta:** la acción solicitada y el recurso.
- **HTTP/1.1:** versión del protocolo.
- **405:** código de respuesta.
- **21871:** bytes reportados en el log; su interpretación exacta depende del formato de logging.
- **Referer:** página declarada como origen de la navegación; puede faltar o falsificarse.
- **User-Agent:** cadena declarada por el cliente; **no es identidad confiable**.

**Correlación:** cotejar hora, ruta, método, IP y resultado entre el equipo de prueba y el servidor. Esto distingue actividad autorizada de actividad a investigar.

Patrones que ameritan triage: numerosas rutas poco habituales en pocos segundos; solicitudes a `/wp-login.php`, `/wp-admin/`, `/xmlrpc.php` en un sitio que no usa WordPress; muchos intentos fallidos; cambios de método o volumen. **No atribuir malicia por un único GET**. Un `200` uniforme con HTML idéntico puede producir falsos positivos.

**Triage SOC N1:** identificar evento → establecer si era esperado → contextualizar activo y origen → confirmar alcance e impacto → guardar evidencias → escalar o cerrar justificadamente.

## 6. Phishing e ingeniería social (TryHackMe — Within Range)

**Ingeniería social:** manipular decisiones humanas mediante confianza, familiaridad, urgencia o autoridad.  
**Phishing:** mensajes que suplantan entidades para provocar clics, entrega de credenciales o acciones indebidas. MITRE ATT&CK **T1566 (Phishing)** describe una técnica utilizada para obtener acceso inicial en ciertos escenarios.

Tres señales identificadas en la simulación:
1. **Remitente / dominio:** que el mensaje mencione una compañía o persona no verifica al remitente.
2. **Enlace:** comparar destino real con el servicio legítimo, sin confiar en el texto visible.
3. **Urgencia artificial:** «actúa en 30 minutos» busca impedir que la víctima verifique.

**Qué hacer:** no abrir enlaces del mensaje sospechoso; ingresar por la app o el dominio oficial; reportar el correo mediante el canal institucional. El SOC puede investigar cabeceras, URL, alcance de la campaña y usuarios potencialmente afectados.

**Exposición de datos del navegador:** un sitio puede inferir/recibir cierta información como idioma, zona horaria, navegador y características del dispositivo. **La ubicación precisa no es automática** y normalmente depende de permisos u otras fuentes. La actividad del teléfono fue una **simulación**, no una interceptación demostrada.

**Resultado formativo observado:** ejercicio Within Range completado, 5/5 señales, 3/3 indicadores, elección Blue Team.

## 7. Glosario rápido

- **Superficie de ataque:** activos, interfaces y rutas por los que puede producirse una interacción maliciosa.
- **Vulnerabilidad:** debilidad explotable bajo determinadas condiciones, no simplemente una ruta encontrada.
- **Autenticación:** comprobar quién eres.
- **Autorización:** comprobar qué puedes hacer.
- **Broken Access Control:** fallo que permite acciones o acceso fuera de permisos.
- **IOC:** indicador de compromiso; dato que puede ayudar en una investigación, nunca prueba concluyente aislada.
- **TTP:** tácticas, técnicas y procedimientos de un adversario.
- **SIEM:** plataforma para centralizar/correlacionar eventos de seguridad.
- **Wazuh:** plataforma de monitoreo y análisis que puede servir para ejercicios SIEM/XDR.
- **MITRE ATT&CK:** base de conocimiento de comportamientos adversarios.
- **Falso positivo:** alerta/hallazgo que no corresponde a la actividad dañina supuesta.

## 8. Qué repasar y practicar después

1. Reconocer sin ayuda la diferencia entre `403`, `404` y `405`.
2. Explicar por qué `200` + HTML idéntico no demuestra que una ruta exista.
3. Leer una línea Apache e identificar IP, fecha, método, URI, estado y User-Agent.
4. Distinguir **autenticación** de **autorización**.
5. Identificar las tres señales de un correo phishing.
6. En el laboratorio aislado: importar logs de ejemplo en Wazuh, generar una alerta de enumeración y justificar si es verdadero o falso positivo.

> Nota de actualización: **NIST CSF 2.0 tiene seis funciones**: Govern, Identify, Protect, Detect, Respond y Recover. Documentos antiguos que mencionan cinco reflejan versiones previas.

**Regla de estudio:** cada comando debe tener un propósito explícito: qué buscamos, por qué, qué respuesta esperamos, qué alternativas explican el resultado y qué hacemos después.
