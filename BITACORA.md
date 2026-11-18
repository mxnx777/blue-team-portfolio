# Bitácora de avance

Registro cronológico **append-only**: las entradas nuevas se agregan al final y no se editan las anteriores. Sirve para mostrar constancia y trazabilidad.

## Cómo crece este portafolio

| Cuando hago… | Va a… | Formato |
|---|---|---|
| Un room de TryHackMe / BTLO | `06-formacion/tryhackme/` (o `btlo/`) + fila en su índice | Guía: conceptos, comandos, errores y preguntas de repaso. **Sin flags ni respuestas literales** |
| Un room de investigación (logs, SIEM, phishing, alertas) | Además, a `01-casos-soc/` | [Plantilla de caso SOC](01-casos-soc/PLANTILLA-CASO-SOC.md) |
| Un ejercicio en BLUELAB | `02-laboratorio/` con el siguiente número | Guía del lab |
| Un ejercicio en BLUELAB con ataque + detección | Además, a `01-casos-soc/` y al catálogo de `03-detecciones/` | Caso + fila de detección con su estado real |
| Cualquiera de los anteriores | Una línea al final de esta bitácora | `AAAA-MM-DD — qué se hizo → dónde quedó` |

---

## Registro

- 2026-07-29 — Inicio del laboratorio BLUELAB: DC Windows Server 2022, cliente Windows 11, red aislada → [lab 01](02-laboratorio/01-bluelab-active-directory.md)
- 2026-08-26 — Auditoría Kerberos en el DC corregida por GPO; aparece el 4771 → [LAB-SOC-2026-003](01-casos-soc/LAB-SOC-2026-003-fallo-kerberos-4771-brecha-visibilidad.md)
- 2026-08-27 — Wazuh manager desplegado, agente Windows + Sysmon, primeros triages → [LAB-SOC-2026-004](01-casos-soc/LAB-SOC-2026-004-triage-alertas-wazuh-falsos-positivos.md)
- 2026-10-30 — Kali vs. endpoint: puerto → PID → servicio → firewall; DROP confirmado → [LAB-SOC-2026-002](01-casos-soc/LAB-SOC-2026-002-escaneo-smb-listen-vs-filtered.md)
- 2026-11-03 — Caso de reconocimiento web con logs Apache → [SOC-001](01-casos-soc/SOC-001-reconocimiento-web-apache.md)
- 2026-11-03 — TryHackMe Defensive Security Intro: informe SOC simulado → [LAB-SOC-2026-001](01-casos-soc/LAB-SOC-2026-001-fuerza-bruta-login-tryhackme.md)
- 2026-11-04 — Autoevaluación de Linux → [06-formacion/linux-autoevaluacion](06-formacion/linux-autoevaluacion/)
- 2026-11-16 — Índice de guías TryHackMe → [06-formacion/tryhackme](06-formacion/tryhackme/)
- 2026-11-18 — Creación del portafolio público: material migrado y ordenado, experiencia laboral anonimizada, 3 casos SOC nuevos a partir del laboratorio, catálogo de detecciones
