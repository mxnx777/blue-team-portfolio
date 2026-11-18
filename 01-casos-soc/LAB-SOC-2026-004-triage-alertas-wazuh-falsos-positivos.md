# LAB-SOC-2026-004 — Triage de alertas Wazuh: Account Discovery y File Deletion generadas por el propio agente

**Fecha:** 2026-08-27  
**Entorno:** laboratorio BLUELAB, Wazuh manager (Ubuntu Server 24.04) + agente en `LAB-CLIENT01` con Sysmon  
**Fuente de la alerta:** Wazuh (reglas 92031 y 92021) sobre Sysmon Event ID 1  
**Severidad asignada:** Informativa tras el triage  
**Estado:** concluido, clasificado como benigno esperado

## 1. Resumen

Dos alertas mapeadas a MITRE ATT&CK (Account Discovery y File Deletion) se investigaron reconstruyendo la cadena de procesos padre-hijo. Ambas tenían como origen el **propio agente Wazuh** ejecutando comprobaciones de configuración como `SYSTEM`. Se clasificaron como **benigno esperado**, dejando documentado qué se validaría antes de cerrarlas en un entorno real.

## 2. Alertas iniciales

| Alerta | rule.id | Nivel | MITRE |
|---|---|---|---|
| Discovery activity executed | 92031 | 3 | T1087 Account Discovery |
| Powershell was used to delete files or directories | 92021 | 3 | T1070.004 File Deletion |

## 3. Evidencia

### Alerta 92031: `net user`

| Campo | Valor |
|---|---|
| Image | `C:\Windows\SysWOW64\net1.exe` |
| CommandLine | `C:\WINDOWS\system32\net1 user` |
| ParentImage | `C:\Windows\SysWOW64\net.exe` (`net user`) |
| User | `NT AUTHORITY\SYSTEM` |
| CurrentDirectory | `C:\Program Files (x86)\ossec-agent\` |

### Alerta 92021: PowerShell con eliminación de archivo

| Campo | Valor |
|---|---|
| Image | `powershell.exe` (SysWOW64) |
| ParentImage | `C:\Program Files (x86)\ossec-agent\wazuh-agent.exe` |
| User / ParentUser | `NT AUTHORITY\SYSTEM` |

```powershell
secedit /export /cfg $env:TEMP\secpol.cfg;
Get-Content $env:TEMP\secpol.cfg | Select-String "ResetLockoutCount";
Remove-Item $env:TEMP\secpol.cfg
```

**Qué hizo realmente:** exportó la política de seguridad local, leyó el valor `ResetLockoutCount` (política de bloqueo de cuentas) y borró el archivo temporal. El `Remove-Item` es lo que disparó la regla de File Deletion.

## 4. Análisis

| | 92031 | 92021 |
|---|---|---|
| **Verificado** | Se ejecutó `net user` (enumeración de cuentas) | PowerShell eliminó un archivo temporal |
| **Contexto** | `SYSTEM`, directorio del agente Wazuh | Padre directo: `wazuh-agent.exe`, como `SYSTEM` |
| **Clasificación** | Benigno esperado (inferencia razonable) | Benigno esperado (contexto fuerte) |

**Por qué no se cierra solo por el nombre de la regla.** Antes de cerrar en un entorno real se validaría:
1. Cadena completa de procesos padre.
2. Hora exacta y qué tarea del agente la originó.
3. Si ocurre igual en otros endpoints (patrón de flota versus anomalía puntual).
4. Que ninguna cuenta humana haya ejecutado el comando.
5. Eventos antes y después (se usó *View surrounding documents* en Wazuh).
6. Ausencia de persistencia, movimiento lateral o escalamiento.

**Riesgo a tener en cuenta:** un atacante que ejecute comandos desde el contexto del agente heredaría esta apariencia de legitimidad. Por eso importan el padre exacto y la línea de comandos completa, no solo el usuario `SYSTEM`.

## 5. MITRE ATT&CK

- T1087 y T1070.004: **comportamiento observado, sin intención maliciosa**. La regla funcionó correctamente; el contexto la clasifica.

## 6. Detección (afinamiento)

Para reducir ruido sin perder visibilidad, una excepción **acotada** (no global), **pendiente de implementar y validar**:

```text
Suprimir 92021 SOLO SI:
  parentImage = C:\Program Files (x86)\ossec-agent\wazuh-agent.exe
  Y user = NT AUTHORITY\SYSTEM
  Y commandLine contiene "secedit /export"
```

Nunca excluir por `user = SYSTEM` a secas: eso abriría un punto ciego enorme.

## 7. Respuesta

Sin acciones de contención: actividad legítima. Se documenta para que el siguiente analista de turno no repita el mismo triage.

## 8. Lecciones

- **Detección válida no es incidente confirmado.** El trabajo del analista N1 es ponerle contexto a la alerta.
- La misma técnica ATT&CK aparece en actividad maliciosa y en tareas legítimas: decide la cadena padre-hijo.
- Un evento puede existir en `archives.json` sin generar alerta en `wazuh-alerts-*`: hay que saber dónde buscar cuando no hay alerta.

Guía completa: [lab 10 — Wazuh SIEM](../02-laboratorio/10-wazuh-siem-agente-windows-sysmon.md)
