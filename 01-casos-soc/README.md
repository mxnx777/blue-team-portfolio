# Casos SOC

Cada caso sigue la [plantilla estándar](PLANTILLA-CASO-SOC.md). Los casos nuevos se agregan **al final** de la tabla.

| ID | Fecha | Caso | Origen | MITRE | Habilidad SOC principal |
|---|---|---|---|---|---|
| [SOC-001](SOC-001-reconocimiento-web-apache.md) | 2026-11-03 | Reconocimiento web y correlación con logs Apache | Prueba autorizada, sanitizada | T1595.003 | Análisis de logs web a escala (14.516 registros) |
| [LAB-SOC-2026-001](LAB-SOC-2026-001-fuerza-bruta-login-tryhackme.md) | 2026-11-03 | Fuerza bruta contra login | TryHackMe (simulación) | T1110.001 | Flujo N1 completo + threat intelligence |
| [LAB-SOC-2026-002](LAB-SOC-2026-002-escaneo-smb-listen-vs-filtered.md) | 2026-10-30 | Escaneo SMB: LISTEN en el endpoint, `filtered` en Nmap | Laboratorio BLUELAB | T1046 | Correlación red ↔ endpoint ↔ firewall |
| [LAB-SOC-2026-003](LAB-SOC-2026-003-fallo-kerberos-4771-brecha-visibilidad.md) | 2026-08-26 | Fallo Kerberos invisible: brecha de auditoría en el DC | Laboratorio BLUELAB | T1110.001 | Detectar y corregir brechas de visibilidad |
| [LAB-SOC-2026-004](LAB-SOC-2026-004-triage-alertas-wazuh-falsos-positivos.md) | 2026-08-27 | Triage de alertas Wazuh: falsos positivos del propio agente | Laboratorio BLUELAB | T1087 · T1070.004 | Triage y clasificación de alertas SIEM |

## Próximos casos (en preparación)

| Caso | Base ya hecha | Qué falta |
|---|---|---|
| Reconocimiento de AD vía LDAP con cuenta estándar | [lab 12](../02-laboratorio/12-reconocimiento-ad-ldap-cuenta-estandar.md): enumeración de usuarios, grupos y SPN con `ldapsearch` | Repetir con auditoría del DC activa y ver qué deja en Security/Wazuh (por validar) |

## Convención de IDs

- `SOC-NNN` → prácticas sobre sistemas reales con autorización (siempre sanitizadas).
- `LAB-SOC-AAAA-NNN` → laboratorio propio o plataformas de práctica.
