# Bases MITRE para analistas SOC N1 — guía de estudio en español

> Fecha de incorporación: 2026-11-03. Material de formación y referencia; **no es un laboratorio ejecutado ni una certificación**.
>
> Fuente principal: Kathryn Knerler, Ingrid Parker y Carson Zimmerman, *11 Strategies of a World-Class Cybersecurity Operations Center*, MITRE, 2.ª ed. (2022).
> PDF oficial: https://www.mitre.org/sites/default/files/2022-04/11-strategies-of-a-world-class-cybersecurity-operations-center.pdf
>
> Resumen interpretativo en español. No pretende reemplazar el libro ni atribuir a MITRE nuestras prioridades personales de formación.

## 1. Objetivo de aprendizaje

Formar criterio para ejercer tareas de **SOC N1** (primer nivel de análisis): recibir una alerta, reunir evidencia, distinguir una actividad normal de una sospechosa, determinar prioridad, escalar cuando corresponda y documentar. El objetivo del portafolio es demostrar *qué se verificó y por qué se concluyó algo*, no acumular comandos sueltos.

El libro estudia cómo se diseña y opera un SOC completo: también incluye gestión, estructura, personal, métricas y responsabilidades que superan un puesto N1. Nuestra prioridad práctica inicial es **respuesta a incidentes, fuentes de datos, herramientas del analista e inteligencia de amenazas**.

## 2. Las 11 estrategias del libro

1. **Conocer qué se protege y por qué**: misión, activos, sistemas críticos, usuarios, amenazas y obligaciones.
2. **Dar autoridad al SOC para realizar su trabajo**: definir mandato, alcance, responsabilidades y acuerdos.
3. **Diseñar una estructura adecuada**: funciones, tamaño, cobertura y relación con otras áreas.
4. **Contratar y desarrollar personal competente**: formación, cultura, turnos y crecimiento.
5. **Priorizar la respuesta a incidentes**: criterios de clasificación, procedimientos, escalamiento y playbooks.
6. **Comprender al adversario mediante inteligencia de amenazas**: contexto, comportamientos y TTP.
7. **Seleccionar y recoger datos adecuados**: telemetría útil, cobertura, calidad, retención y costos.
8. **Usar herramientas para facilitar el trabajo analítico**: búsquedas, correlación, casos y automatización.
9. **Comunicar, colaborar y compartir**: informes claros, coordinación y traspasos.
10. **Medir el desempeño para mejorar**: métricas que ayuden a evaluar resultados y detectar brechas.
11. **Ampliar progresivamente las capacidades del SOC**: hunting, ingeniería de detección y otras funciones especializadas.

**Orden de estudio propuesto (decisión pedagógica nuestra, no orden obligatorio MITRE):** 5 → 7 → 8 → 6, intercalando los fundamentos de 1, 2, 3 y 9. Dejar gestión avanzada y capacidades especializadas para después.

## 3. Conceptos que deben distinguirse

| Concepto | Significado práctico | Ejemplo |
|---|---|---|
| Evento | Hecho registrado por un sistema. | Windows registra un inicio de sesión. |
| Alerta | Señal generada por una regla, herramienta o analista que requiere revisión. | Wazuh marca varios fallos de acceso. |
| Incidente | Suceso con impacto real o potencial sobre la seguridad, evaluado según criterios organizacionales. | Acceso no autorizado confirmado. |
| Falso positivo | Alerta que, tras analizarla, no corresponde a la actividad maliciosa que sugería la detección. | Mantenimiento autorizado detectado como sospechoso. |
| Triage | Evaluación inicial: validez, contexto, severidad, prioridad y siguiente acción. | Identificar cuenta, IP, hora, host e historial. |
| Correlación | Relacionar evidencias de varias fuentes en un contexto temporal. | Evento 4625 + IP + firewall + eventos posteriores. |
| IOC | Indicador de compromiso: evidencia o artefacto asociado a posible intrusión. | Hash, dominio o IP, *siempre contextualizados*. |
| TTP | Tácticas, técnicas y procedimientos: **cómo actúa** un adversario. | Ejecución de comandos y movimiento lateral. |
| Playbook | Procedimiento documentado para una situación específica. | Pasos de análisis de autenticación sospechosa. |
| Escalamiento | Transferencia fundamentada de un caso a alguien con atribuciones/conocimientos adicionales. | Un N1 remite a N2 con hallazgos y evidencia. |

**Regla de criterio:** evento ≠ alerta ≠ incidente. Un HTTP 200 no demuestra necesariamente que un archivo exista; el sitio puede responder con una página genérica. Un HTTP 405 al usar DELETE demuestra rechazo del método en esa solicitud, **no prueba por sí solo toda la seguridad del servidor**.

## 4. Siglas indispensables (inglés → español → aplicación)

| Sigla | Inglés | Español / uso |
|---|---|---|
| SOC | Security Operations Center | Centro de operaciones de seguridad. |
| CSOC | Cybersecurity Operations Center | SOC centrado en ciberseguridad. |
| N1 / L1 | Level 1 | Primer nivel de análisis, validación y escalamiento. |
| SIEM | Security Information and Event Management | Centralización, búsqueda y correlación de registros/alertas. |
| EDR | Endpoint Detection and Response | Telemetría, detección y respuesta en equipos finales. |
| XDR | Extended Detection and Response | Detección/respuesta integrando varias fuentes. |
| CTI | Cyber Threat Intelligence | Inteligencia sobre amenazas. |
| IOC | Indicator of Compromise | Indicador de posible compromiso. |
| TTP | Tactics, Techniques and Procedures | Patrones de comportamiento adversario. |
| IR | Incident Response | Respuesta a incidentes. |
| DFIR | Digital Forensics and Incident Response | Informática forense y respuesta a incidentes. |
| SOAR | Security Orchestration, Automation and Response | Orquestación/automatización de tareas de seguridad y respuesta. |
| SOP | Standard Operating Procedure | Procedimiento operativo estándar. |
| ATT&CK | Adversarial Tactics, Techniques, and Common Knowledge | Base de conocimiento de tácticas/técnicas adversarias de MITRE; **no es el libro de 11 estrategias**. |
| OODA | Observe, Orient, Decide, Act | Observar, orientar, decidir, actuar; modelo de toma de decisiones. |
| DNS | Domain Name System | Resolución de nombres de dominio. |
| HTTP | Hypertext Transfer Protocol | Protocolo de solicitudes/respuestas web. |
| PID | Process Identifier | Identificador numérico de un proceso. |
| GPO | Group Policy Object | Objeto de directiva de grupo de Windows/AD. |
| AD | Active Directory | Servicio de directorio, identidades y administración de dominio. |
| MITRE | Nombre propio de la organización MITRE | No lo tratar como si fuera la sigla de ATT&CK. |

**Cómo estudiar una sigla:** nombre completo, traducción clara, para qué sirve, ejemplo observable y cómo la vería un analista. No memorizar letras sin entender la función.

## 5. Flujo práctico de SOC N1

1. **Recibir alerta**: fuente, hora y regla que la generó.
2. **Validar señal**: comprobar si existe evidencia real y si hubo errores de detección.
3. **Enriquecer contexto**: usuario, host, IP, procesos, eventos anteriores y posteriores, criticidad del activo.
4. **Formular hipótesis**: principal y alternativas; marcar las que todavía no se prueban.
5. **Clasificar y priorizar**: probabilidad, posible impacto, alcance y urgencia según procedimiento.
6. **Decidir**: cerrar justificadamente, continuar investigación o escalar; no ejecutar contenciones fuera de autoridad.
7. **Documentar**: línea de tiempo, fuentes, consultas, resultados, decisión, limitaciones y próximo paso.

### Plantilla mínima de informe

- **Identificador de caso**:
- **Fecha/hora y zona horaria**:
- **Alerta original / regla**:
- **Sistema y usuario involucrados**:
- **Hipótesis principal / alternativas**:
- **Fuentes consultadas**:
- **Eventos o indicadores verificados**:
- **Línea de tiempo**:
- **Impacto y alcance conocidos / desconocidos**:
- **Decisión y justificación**:
- **Escalamiento / acciones pendientes**:
- **Evidencias sanitizadas**:

## 6. Cómo aplicaremos estas bases al laboratorio

| Caso | Evidencia esperada | Competencia |
|---|---|---|
| Accesos a Windows | 4624, 4625, 4672; IP, cuenta, Logon ID y horarios | Triage de autenticación. |
| PowerShell codificado | Sysmon Event ID 1, línea de comandos, procesos padre/hijo y decodificación en entorno controlado | Análisis de ejecución y ATT&CK. |
| Conexiones de red | Sysmon Event ID 3, SourceIp, DestinationIp, DestinationPort, ProcessGuid | Correlación proceso-red. |
| Solicitudes web | Logs de Apache: GET, HEAD, DELETE, respuesta 200/405, bytes, User-Agent | Interpretación de actividad HTTP y limitaciones de evidencia. |
| Alertas SIEM | Regla Wazuh, datos de origen, contexto, decisión y ticket | Flujo completo N1. |

**Importante:** estos son casos de estudio y objetivos didácticos. Un caso no se declarará «completado» sin evidencia observada y documentación verificada.

### Diferenciar teoría / evidencia / decisión

- **Teoría:** MITRE recomienda disponer de procedimientos claros de respuesta.
- **Evidencia:** un log prueba que una solicitud llegó y qué respuesta registró el servidor.
- **Inferencia:** esa solicitud *podría* indicar una actividad sospechosa; se investiga antes de etiquetarla.
- **Decisión:** se cierra, escala o contiene conforme al alcance y autorización.

## 7. Hoja de ruta incremental

- [ ] Comprender evento, alerta, incidente, triage y escalamiento.
- [ ] Interpretar correctamente registros Windows y Apache.
- [ ] Correlacionar evento, proceso, usuario, IP y tiempo.
- [ ] Investigar una alerta real o simulada en Wazuh.
- [ ] Relacionar una técnica observada con MITRE ATT&CK sin forzar atribuciones.
- [ ] Elaborar un informe N1 con evidencia y límites explícitos.
- [ ] Repetir hasta lograr autonomía y reproducibilidad.

**Norma para futuras sesiones:** explicar cada sigla en su primera aparición; indicar **qué buscamos, por qué lo buscamos, qué resultado esperamos y qué significa realmente**. Avanzar una verificación por vez y no repetir una prueba si no aporta evidencia nueva.

## 8. Fuentes y alcance

- MITRE, [11 Strategies of a World-Class Cybersecurity Operations Center (2022)](https://www.mitre.org/sites/default/files/2022-04/11-strategies-of-a-world-class-cybersecurity-operations-center.pdf) — referencia sobre operación de SOC.
- [MITRE ATT&CK](https://attack.mitre.org/) — base de conocimientos de tácticas y técnicas, recurso distinto del libro.
- [NIST Cybersecurity Framework 2.0](https://www.nist.gov/cyberframework) — marco complementario con seis funciones: Govern, Identify, Protect, Detect, Respond, Recover.
- Prioridades SOC N1, ejemplos de Windows/Sysmon/Apache y hoja de ruta: **adaptación pedagógica del proyecto**, no afirmaciones textuales del libro.

**Criterio de éxito:** poder explicar qué pasó, cómo se sabe, qué falta por verificar y qué debería hacer un SOC N1.
