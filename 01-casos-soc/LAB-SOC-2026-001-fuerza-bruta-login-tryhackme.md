# LAB-SOC-2026-001 — Informe SOC de laboratorio (TryHackMe)

**Fecha:** 2026-11-03  
**Entorno:** FakeBank, escenario ficticio de TryHackMe Defensive Security Intro  
**Clasificación en la simulación:** Critical  
**Estado:** informe enviado y confirmado  
**Código de portafolio:** LAB-SOC-2026-001

## Resumen
En un escenario SOC simulado, se investigó un ataque de adivinación de contraseñas contra una página de inicio de sesión. La plataforma identifica al actor ficticio como ShadowFigures. Se realizó triage, se revisó la ficha de inteligencia de amenazas y se envió un informe.

## IDs y nombres: no inventarlos
- **LAB-SOC-2026-001:** convención local para el portafolio, LAB-AREA-AÑO-CORRELATIVO.
- **SOC-2026-1103-001:** referencia interna que aparece en el informe del simulador.
- **SEC-2026-341:** confirmación que emite el simulador al enviar el reporte.

Un SOC real utiliza los IDs de su sistema de tickets o SIEM y conserva las referencias cruzadas.

## Evidencia observada
| Campo | Dato de simulación |
|---|---|
| Método | Password guessing / fuerza bruta |
| Actor | ShadowFigures (ficticio) |
| Página objetivo | /login |
| Usuario objetivo | Cuenta ficticia registrada en el simulador |
| Hora del reporte | 2026-11-03 14:44 UTC |
| Contención | IP bloqueada a nivel perimetral y cuenta marcada/bloqueada |
| Analistas | Analyst y Cassidy |
| Resultado | Report Submitted |

**Límites:** no se proporcionaron registros crudos de autenticación, número exacto de intentos, evidencia de acceso exitoso ni prueba de exfiltración. La hora 11:42 del formulario de inteligencia no tiene zona horaria indicada; no debe equipararse automáticamente a la del informe.

## MITRE ATT&CK
- **T1110.001 Password Guessing:** técnica relacionada directamente con el escenario.
- **T1110.004 Credential Stuffing, T1078 Valid Accounts, T1133 External Remote Services y T1090 Proxy:** capacidades mencionadas en la ficha ficticia del actor, **no demostradas como realizadas en este evento**.

## Flujo SOC N1
1. **Detección:** registrar alerta, hora y origen de la telemetría.
2. **Triage:** identificar cuenta, servicio, frecuencia de fallos y posibles inicios exitosos.
3. **Contexto:** consultar inteligencia de amenazas sin asumir atribución solo por coincidencia de IP.
4. **Contención:** coordinar medidas urgentes cuando corresponda, en paralelo con conservación de evidencia.
5. **Investigación:** determinar alcance en otras cuentas y sistemas; distinguir confirmado de hipotético.
6. **Informe:** documentar observaciones, decisiones, responsables, evidencias y números de ticket.
7. **Seguimiento:** validar recuperación y monitorear recurrencias antes del cierre.

## Qué aprendí
- Contención no es necesariamente erradicación.
- Un indicador de compromiso no demuestra por sí solo quién atacó.
- El historial de técnicas de un actor no significa que todas ocurrieron en este caso.
- Un informe enviado no equivale a incidente resuelto.

## Plantilla reutilizable
| Campo | Completar en futuras investigaciones |
|---|---|
| ID local | LAB-SOC-AAAA-NNN |
| Ticket original | Identificador emitido por la plataforma |
| Fuente de alerta | SIEM, EDR, servidor o firewall |
| Fecha y zona | ISO 8601 |
| Sistemas y cuentas | Objetivos realmente observados |
| Evidencia | Logs, capturas, hashes cuando correspondan |
| Hipótesis | Principal y alternativas |
| Técnica MITRE | Solo las demostradas |
| Impacto y alcance | Confirmado / pendiente |
| Medidas | Contención, erradicación y recuperación |
| Validación de cierre | Resultado y evidencia |

**Nota:** Caso académico de práctica guiada. No se incluyen datos productivos ni archivos privados.

**Siguiente ejercicio:** reproducir actividad controlada de inicio de sesión en .BLUELAB y analizar telemetría en Wazuh.
