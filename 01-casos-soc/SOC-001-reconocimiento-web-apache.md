# Caso SOC-001 — Reconocimiento web y correlación con Apache

**Fecha:** 2026-11-03  
**Entorno:** sitio web institucional administrado, pruebas controladas con Kali y revisión de registros Apache/cPanel  
**Enfoque:** Blue Team / SOC N1 · HTTP · reconocimiento web · validación de hallazgos  
**Estado:** análisis concluido; remediaciones pendientes de verificación  
**Divulgación:** versión sanitizada. No contiene registros brutos, IP públicas, credenciales, nombres de archivos institucionales sensibles ni claves API.

## Objetivo

Demostrar la cadena de evidencias desde una acción de reconocimiento web hasta la detección en el registro de acceso del servidor, distinguiendo:
- ruta descubierta vs. vulnerabilidad confirmada;
- lectura pública vs. autorización para modificación;
- respuesta HTTP 200 vs. recurso realmente existente;
- actividad de auditoría autorizada vs. tráfico externo por investigar.

## Alcance y método

1. Reconocimiento autorizado de un sitio web con **DIRB** y un diccionario común.
2. Validación de respuestas seleccionadas mediante **curl** y navegador.
3. Creación de un archivo **desechable**, sin datos privados, en un área publicada para demostrar acceso HTTP.
4. Solicitud de lectura GET, seguida de una solicitud HTTP DELETE limitada al archivo desechable.
5. Revisión de cPanel (índices y document root) y descarga del **access log** de Apache.
6. Correlación temporal de solicitudes cliente-servidor y análisis de patrones agregados.

**Nota operativa:** ninguna prueba mostró eliminación remota exitosa. No se interpretó el 200 como éxito de autenticación.

## Evidencia cuantitativa

| Indicador | Resultado |
|---|---:|
| Registros HTTP analizados | 14.516 |
| Solicitudes asociadas a la IP de pruebas | 13.945 (96,1 %) |
| Solicitudes identificables del reconocimiento automatizado | 13.929 |
| Ventana del reconocimiento | 09:30:30–09:32:53 UTC−03 |
| Tasa media aproximada durante esa ventana | 97 solicitudes/s |
| Solicitudes de otras IP | 571 |
| Respuestas 404 | 9.219 |
| Respuestas 200 | 5.197 |
| Respuestas 403 | 84 |
| Respuestas 301 | 9 |
| Respuestas restantes | 7 |
| Respuestas de 21.871 bytes | ~14.302 |

El **User-Agent** del reconocimiento declaraba ser Internet Explorer antiguo; es una cadena controlada por el cliente y no una prueba de identidad.

## Hallazgos comprobados

### H-01 — Listado de directorios habilitado

Se observó una respuesta HTML con `Index of` en una carpeta de recursos estáticos. **Impacto confirmado:** visibilidad pública del listado, nombres y metadatos de archivos publicados. **Impacto no demostrado:** exposición de información sensible o capacidad de modificación.

**Tratamiento recomendado:** evaluar `No Indexing` en cPanel para las carpetas correspondientes, considerar herencias y excepciones, y validar la publicación normal del sitio.

### H-02 — Copia de sitio web bajo raíz pública

Una copia del sitio fue colocada deliberadamente en la raíz web para simular una publicación accidental. Un archivo marcador desechable se pudo leer por HTTP sin iniciar sesión.

**Interpretación:** un archivo dentro de la raíz pública puede ser consultado si se conoce su URL. Esto no demuestra acceso indebido por sí solo. Un respaldo real dentro de esa raíz puede elevar el riesgo según su contenido.

**Tratamiento recomendado:** retirar la copia de prueba de la raíz pública al concluir el ejercicio; guardar respaldos auténticos fuera del directorio servido por HTTP.

### H-03 — Método DELETE rechazado

La solicitud HTTP `DELETE` dirigida al archivo marcador obtuvo **405 Method Not Allowed** y el archivo siguió presente en el gestor de archivos del hosting.

**Conclusión acotada:** este método no permitió eliminar ese recurso; no se evaluaron otros posibles fallos de autorización ni se concluyó seguridad total del sitio.

### H-04 — Respuestas repetidas que dificultan la enumeración

Miles de solicitudes retornaron cuerpos de **21.871 bytes**. Algunas rutas inexistentes devolvían el HTML institucional con HTTP 200.

**Interpretación:** posibles respuestas predeterminadas, reescrituras o reglas de manejo de errores; no puede inferirse la existencia de recursos solo por un código 200. Investigar configuración del servidor y diferenciar recursos existentes de respuestas genéricas.

## Tráfico externo observado

Se detectaron solicitudes a rutas típicas de WordPress, XML-RPC y metadatos Git; también rutas con nombres HTML aparentemente aleatorios. Se omitieron IP y rutas específicas de prueba por higiene de publicación.

**Criterio de investigación:** correlacionar origen, cantidad de peticiones, rutas distintas, franjas temporales y telemetría de aplicación/WAF. No se encontró evidencia de acceso administrativo exitoso en este access log, pero el registro por sí solo no puede descartar incidentes.

## Propuesta de detección SOC

**Hipótesis:** una enumeración automatizada genera una combinación anómala de frecuencia y diversidad de rutas solicitadas.

Lógica de laboratorio (por validar, no activar como bloqueo en producción):

```text
Agrupar por IP de origen durante 60 s
SI total_solicitudes >= 300
Y rutas_unicas >= 150
ENTONCES alerta: posible reconocimiento web
Enriquecer con métodos HTTP, códigos, user-agent y contexto de pruebas autorizadas
```

Las cifras son **umbrales iniciales de ejercicio**, no una regla universal. Ajustar a tráfico legítimo y medir falsos positivos.

## Competencias demostradas

- Uso de DIRB y curl para comprobar hipótesis distintas.
- Lectura de registros Apache y correlación de IP, hora, método, ruta y estado.
- Interpretación de 200 / 403 / 404 / 405, incluidos falsos positivos.
- Diferenciación entre autorización, reconocimiento y compromiso.
- Priorización de remediaciones y documentación auditable.

## Limitaciones y pendientes

- Ventana de logs de aproximadamente dos horas, insuficiente como línea base histórica.
- Sin correlación con logs de aplicaciones, autenticación, WAF ni firewall.
- Verificar configuración Apache de errores/rewrites antes de atribuir causas.
- Confirmar retiro de la copia de prueba, deshabilitación de índices donde corresponda y validación posterior.
- Integrar una muestra **sintética** de estos eventos en el laboratorio Wazuh, no publicar el registro productivo completo.

**Resultado final:** práctica Blue Team documentada, con evidencia de reconocimiento y respuesta del servidor, **sin evidencia de borrado remoto exitoso**.
