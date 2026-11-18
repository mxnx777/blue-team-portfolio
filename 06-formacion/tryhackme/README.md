# Índice de estudio — TryHackMe · Blue Team / SOC N1

**Actualizado:** 16 de noviembre de 2026 · **Rama:** `main`

Este directorio reúne las **guías detalladas ya subidas al repositorio**. Se separan las prácticas realmente ejecutadas de los ejemplos propuestos para practicar más adelante. Cada documento incluye explicaciones, siglas, errores y preguntas para repaso.

| Orden | Guía | Lo que contiene |
|---:|---|---|
| 1 | [Active Directory: OU, delegación, GPO, Kerberos y relaciones de confianza](active-directory-ou-delegacion-gpo.md) | Tasks 1–9; diferencia entre cuentas/OU/GPO; configuración y correcciones; tickets TGT/TGS; NetNTLM; árboles, bosques y trusts; preguntas |
| 2 | [Windows CMD y PowerShell para SOC](windows-cmd-powershell-guia-practica-soc.md) | CMD, SSH, puertos, procesos, PID, filtros, pipeline, hashes, servicios, conexiones, `Invoke-Command` y ejercicios |
| 3 | [Linux Shell Scripting: script de búsqueda en logs](linux-shell-scripting-flag-hunt-laboratorio.md) | Script `flag_hunt.sh`; `for`, `grep`, `basename`; depuración de espacios, permisos; evidencia del `authentication.log`; preguntas |
| 4 | [Networking Concepts: OSI y TCP/IP](networking-concepts-osi-tcpip-guia-soc.md) | Siete capas OSI, cuatro TCP/IP, MAC/IP/puertos, SMB, DNS, SQL Server y análisis SOC |
| 5 | [Networking Concepts: IPv4, subredes y enrutamiento](networking-concepts-ipv4-subredes-enrutamiento.md) | Cuatro octetos, máscara `/24`, rangos RFC1918, NAT, router, comandos Windows/Linux, preguntas y respuestas |

## Cómo estudiar sin limitarse a copiar

1. **Leer** la idea central y expandir todas las siglas en voz alta.
2. **Reproducir** comandos únicamente en equipos del laboratorio o con autorización.
3. **Explicar** qué salida esperamos y qué salida apareció.
4. **Separar** lo confirmado de hipótesis, pruebas pendientes y ejemplos propuestos.
5. **Responder** las preguntas sin mirar la clave, y repetir los conceptos que cuesten.

## Estado y límites

- Estas guías **no equivalen a certificaciones** ni garantizan que cada procedimiento se ejecutó en el laboratorio personal `.BLUELAB`.
- La guía de AD distingue configuraciones observadas de validaciones que quedaron pendientes.
- El laboratorio Linux se resolvió sin acceso `root` y con algunos logs inaccesibles; eso está declarado en la guía.
- La Task 4 de Networking Concepts incluye **dos respuestas confirmadas** en capturas; los ejercicios adicionales de subredes son propuestos.
- No publicar credenciales, tokens ni datos sensibles de producción.

**Objetivo del portafolio:** evidencias reproducibles, razonamiento técnico, diagnóstico, comandos explicados y documentación clara.
