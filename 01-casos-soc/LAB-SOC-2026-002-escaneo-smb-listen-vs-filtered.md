# LAB-SOC-2026-002 — Escaneo SMB desde Kali: puerto LISTEN en el endpoint, `filtered` en Nmap

**Fecha:** 2026-10-30  
**Entorno:** laboratorio BLUELAB (`bluelab.test`, red aislada `172.31.250.0/24`)  
**Fuente de la alerta:** resultado de reconocimiento + telemetría del endpoint (`Get-NetTCPConnection`, `pfirewall.log`)  
**Severidad asignada:** Baja (actividad controlada, sin acceso exitoso)  
**Estado:** concluido con evidencia

## 1. Resumen

Un escaneo Nmap desde Kali contra `LAB-CLIENT01` mostró TCP/445 como `filtered`, mientras que en el endpoint el puerto estaba en `LISTEN`. Se investigó la aparente contradicción hasta demostrar, con el log del firewall, que **Windows Firewall descartó (DROP) las sondas**. Conclusión respaldada por evidencia del endpoint, no solo por la herramienta.

## 2. Alerta inicial

| Campo | Valor |
|---|---|
| Origen | Kali `172.31.250.40` |
| Destino | `LAB-CLIENT01` `172.31.250.20` |
| Comando | `nmap -n -Pn -p 445 172.31.250.20` |
| Resultado | `445/tcp filtered microsoft-ds` |

`microsoft-ds` es solo el nombre que Nmap asocia al puerto 445; no indica que se haya establecido una sesión SMB.

## 3. Evidencia

| Paso | Fuente | Comando | Resultado observado |
|---|---|---|---|
| 1 | Endpoint | `Get-NetTCPConnection -LocalPort 445 -State Listen` | `445` LISTEN, `OwningProcess = 4` (System) |
| 2 | Endpoint | `Get-NetFirewallProfile \| Select Name,Enabled,DefaultInboundAction` | Los 3 perfiles habilitados |
| 3 | Endpoint | `Get-NetFirewallRule -Enabled True -Direction Inbound -Action Allow \| Get-NetFirewallPortFilter \| Where {$_.LocalPort -eq 445}` | Sin salida: no hay regla Allow explícita para 445 |
| 4 | Endpoint | `Set-NetFirewallProfile -Profile Domain,Private,Public -LogBlocked True` | Logging de bloqueos activado |
| 5 | Kali | Repetición del `nmap -p 445` | `filtered` |
| 6 | Endpoint | `Get-Content C:\Windows\System32\LogFiles\Firewall\pfirewall.log -Tail 30` | `DROP TCP 172.31.250.40 172.31.250.20 <puerto-efímero> 445` |

## 4. Análisis

- **Hipótesis principal:** Windows Firewall descarta el tráfico entrante a TCP/445.
- **Alternativas consideradas:** firewall de red intermedio, segmentación o ACL. Descartadas: red plana de laboratorio, sin equipos intermedios.
- **Qué la confirmó:** la línea `DROP TCP` del `pfirewall.log`, que coincide en IP de origen, IP de destino, protocolo y puerto con la sonda de Kali.

**Ruido descartado:** en el mismo log aparecían `DROP UDP 172.31.250.10 → 172.31.250.255:138` (broadcast NetBIOS del DC). No corresponden al escaneo; el log mezcla todo el tráfico bloqueado, por eso se correlacionó por hora, IPs, protocolo y puertos.

**Punto clave:** la pregunta "si escucha localmente, ¿por qué aparece filtrado?" se responde así: `LISTEN` prueba que existe un socket local, pero no que una conexión remota pueda atravesar el firewall.

## 5. MITRE ATT&CK

- **T1046 — Network Service Discovery:** demostrada (escaneo de puertos desde Kali).

## 6. Detección

Lógica propuesta para el SIEM, **por validar en el laboratorio**:

```text
Fuente: pfirewall.log del endpoint (ingestado en Wazuh)
Agrupar por IP de origen en una ventana de 60 s
SI hay DROP hacia ≥ 10 puertos distintos del mismo destino
ENTONCES alerta: posible escaneo de puertos interno
```

Falsos positivos esperados: escáneres de vulnerabilidades autorizados y herramientas de inventario. Mantener una lista blanca documentada.

## 7. Respuesta

En el laboratorio no se requiere contención: actividad propia y bloqueada. En un entorno real:
1. Identificar el host de origen y si el escaneo estaba autorizado.
2. Si no lo estaba, aislar el host de origen y revisar qué proceso generó el tráfico.
3. Verificar si otros destinos respondieron `open` (alcance real del reconocimiento).

## 8. Lecciones y pendientes

- No concluir a partir de una sola herramienta: Nmap dice qué ve el atacante; el endpoint explica por qué.
- `svchost.exe` no basta: con `tasklist /svc /FI "PID eq <PID>"` se identifica el servicio real (ejemplo del mismo lab: puerto 5040 → PID 4516 → `CDPSvc`).
- En sistemas en español, buscar reglas de firewall por nombre en inglés falla; es mejor filtrar por puerto.
- **Pendiente:** ingestar `pfirewall.log` en Wazuh y crear la regla de la sección 6.

Guías completas: [lab 11](../02-laboratorio/11-kali-nmap-smb-firewall-drop.md) · [lab 15](../02-laboratorio/15-puertos-pid-servicio-firewall-nmap-smb.md)
