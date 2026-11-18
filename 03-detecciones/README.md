# Detecciones

Catálogo de lógica de detección del laboratorio. Cada fila indica su **estado real**: no se marca como validada ninguna detección que no se haya probado con el ataque que debería dispararla.

## Catálogo

| ID | Detección | Fuente | MITRE | Estado | Caso |
|---|---|---|---|---|---|
| DET-001 | Ejecución de comandos de descubrimiento (`net user`) | Sysmon EID 1 → Wazuh regla 92031 | T1087 | ✅ Validada (regla nativa de Wazuh, disparada en el lab) | [LAB-SOC-2026-004](../01-casos-soc/LAB-SOC-2026-004-triage-alertas-wazuh-falsos-positivos.md) |
| DET-002 | PowerShell eliminando archivos | Sysmon EID 1 → Wazuh regla 92021 | T1070.004 | ✅ Validada (regla nativa) · ⏳ Excepción acotada pendiente | [LAB-SOC-2026-004](../01-casos-soc/LAB-SOC-2026-004-triage-alertas-wazuh-falsos-positivos.md) |
| DET-003 | Escaneo de puertos interno | `pfirewall.log` (DROP) | T1046 | ⏳ Lógica propuesta; evidencia del DROP confirmada, falta ingestarla en Wazuh | [LAB-SOC-2026-002](../01-casos-soc/LAB-SOC-2026-002-escaneo-smb-listen-vs-filtered.md) |
| DET-004 | Password spraying / fuerza bruta en Kerberos | Security del DC, EID 4771 `0x18` | T1110.001 / .003 | ⏳ Evento confirmado; falta la regla de correlación en Wazuh | [LAB-SOC-2026-003](../01-casos-soc/LAB-SOC-2026-003-fallo-kerberos-4771-brecha-visibilidad.md) |
| DET-005 | Reconocimiento web automatizado | Access log de Apache | T1595.003 | ⏳ Lógica propuesta | [SOC-001](../01-casos-soc/SOC-001-reconocimiento-web-apache.md) |

## Configuración de telemetría

| Archivo | Qué es |
|---|---|
| [configuracion/sysmonconfig.xml](configuracion/sysmonconfig.xml) | Configuración de Sysmon del endpoint, con exclusión de ruido DNS validada |
| [configuracion/wazuh-agent-ossec-sysmon.xml](configuracion/wazuh-agent-ossec-sysmon.xml) | Bloque del agente Wazuh para recolectar el canal de Sysmon |

## Campos clave para triage en Wazuh (Sysmon Event ID 1)

```text
data.win.system.channel        = Microsoft-Windows-Sysmon/Operational
data.win.system.eventID        = 1
data.win.eventdata.image           → qué se ejecutó
data.win.eventdata.commandLine     → con qué argumentos
data.win.eventdata.parentImage     → quién lo lanzó
data.win.eventdata.user            → con qué cuenta
data.win.eventdata.hashes          → para consultar reputación
```

Pregunta que guía el triage: **quién → ejecutó qué → desde qué proceso padre → cuándo → ¿es esperado?**

Si un evento no aparece como alerta, buscarlo en los archivos crudos (requiere `archives` habilitado en el manager):

```bash
sudo grep -i "<proceso>" /var/ossec/logs/archives/archives.json | tail -5
```

## Playbook

- [Threat hunting y triage con Wazuh + Sysmon](playbook-threat-hunting-triage-wazuh-sysmon.md)
