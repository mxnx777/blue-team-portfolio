# Kali + Nmap + SMB + Firewall DROP

## Objetivo

Correlacionar una prueba de reconocimiento desde Kali Linux contra LAB-CLIENT01 con la evidencia local de Windows y el log del firewall.

## Topología usada

- Kali Linux: `172.31.250.40`
- LAB-CLIENT01: `172.31.250.20`
- Red de laboratorio: `172.31.250.0/24`

## 1. Verificación local de SMB en Windows

En LAB-CLIENT01 se comprobó que TCP/445 estaba escuchando:

```powershell
Get-NetTCPConnection -LocalPort 445 -State Listen |
Select-Object LocalAddress,LocalPort,OwningProcess
```

Resultado observado:

- LocalPort: `445`
- OwningProcess: `4`

PID 4 corresponde al proceso `System`.

Conclusión parcial: SMB estaba escuchando localmente aunque todavía no se sabía si era accesible desde otro host.

## 2. Prueba desde Kali

Desde Kali se ejecutó:

```bash
nmap -n -Pn -p 445 172.31.250.20
```

Resultado:

```text
445/tcp filtered microsoft-ds
```

Interpretación:

- `microsoft-ds` es el nombre de servicio que Nmap asocia normalmente al puerto TCP/445.
- `filtered` indica que Nmap no pudo determinar si el puerto estaba abierto o cerrado porque sus sondas no recibieron una respuesta útil.
- Esto no contradice que Windows muestre el puerto en LISTEN: un servicio puede escuchar localmente y aun así estar bloqueado desde la red.

## 3. Estado del firewall de Windows

Se revisaron los perfiles:

```powershell
Get-NetFirewallProfile |
Select-Object Name,Enabled,DefaultInboundAction,DefaultOutboundAction
```

Los perfiles Domain, Private y Public estaban habilitados.

Luego se habilitó el registro de tráfico bloqueado:

```powershell
Set-NetFirewallProfile -Profile Domain,Private,Public -LogBlocked True
```

Verificación:

```powershell
Get-NetFirewallProfile |
Select-Object Name,Enabled,LogBlocked,LogFileName
```

Archivo de log:

```text
C:\Windows\System32\LogFiles\Firewall\pfirewall.log
```

## 4. Evidencia definitiva del bloqueo

Después de repetir el Nmap contra TCP/445 desde Kali, se revisó:

```powershell
Get-Content "C:\Windows\System32\LogFiles\Firewall\pfirewall.log" -Tail 30
```

El firewall registró tráfico con esta estructura:

```text
DROP TCP 172.31.250.40 172.31.250.20 <puerto-origen-efimero> 445 ...
```

Esto demuestra:

```text
Kali 172.31.250.40
        |
        | TCP hacia 445
        v
LAB-CLIENT01 172.31.250.20
        |
        +-- TCP/445 LISTENING localmente
        |
        +-- Windows Firewall descarta la sonda
        |
        v
Nmap => filtered
```

## 5. Por qué también aparecen DROP UDP

En el mismo `pfirewall.log` se observaron eventos UDP, por ejemplo tráfico desde `172.31.250.10` hacia `172.31.250.255:138`.

Esto no corresponde al Nmap TCP/445 de Kali.

El firewall registra todo el tráfico bloqueado, por lo que el archivo mezcla distintos eventos y protocolos. En este caso:

- `DROP TCP 172.31.250.40 -> 172.31.250.20:445`: prueba de Kali.
- `DROP UDP 172.31.250.10 -> 172.31.250.255:138`: otro tráfico de Windows/red, asociado a NetBIOS Datagram/broadcast.

Para una investigación SOC se debe correlacionar como mínimo:

- hora;
- IP origen;
- IP destino;
- protocolo;
- puerto origen;
- puerto destino.

## 6. Cadena de investigación aprendida

Una forma útil de razonar ante un puerto es:

```text
Puerto
  -> PID
  -> proceso
  -> servicio
  -> accesibilidad desde red
  -> firewall
  -> evidencia de log
```

Ejemplos utilizados:

```powershell
Get-NetTCPConnection -State Listen
tasklist /FI "PID eq <PID>"
tasklist /svc /FI "PID eq <PID>"
```

Para un `svchost.exe`, identificar el servicio real:

```powershell
Get-CimInstance Win32_Service |
Where-Object {$_.ProcessId -eq <PID>} |
Select-Object Name,DisplayName,State,StartMode,ProcessId
```

## 7. Aprendizaje SOC

No se debe concluir únicamente a partir de una herramienta.

En este caso:

1. Nmap indicó `filtered`.
2. Windows demostró que TCP/445 sí estaba en estado LISTEN.
3. El PID permitió identificar quién poseía el listener.
4. El firewall tenía logging habilitado.
5. El archivo `pfirewall.log` mostró explícitamente el `DROP TCP` originado desde Kali.

Conclusión final:

> TCP/445 estaba escuchando en LAB-CLIENT01, pero Windows Firewall filtró la conexión entrante procedente de Kali. La clasificación `filtered` de Nmap quedó respaldada por evidencia del endpoint.

## Próximo paso

Continuar desde esta base hacia reconocimiento SMB controlado dentro del laboratorio y correlacionar esa actividad con Sysmon/Wazuh.
