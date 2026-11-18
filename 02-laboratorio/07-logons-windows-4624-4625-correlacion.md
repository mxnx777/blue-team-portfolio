# Guía práctica — Logons Windows 4624/4625 y correlación con Kerberos

**Fecha:** 2026-08-26  
**Laboratorio:** `bluelab.test`  
**Cliente:** `LAB-CLIENT01` — `172.31.250.20`  
**Controlador de dominio:** `LAB-DC01` — `172.31.250.10`  
**Usuario observado:** `BLUELAB\adm-mxnx`

## Objetivo

Aprender a interpretar eventos de inicio de sesión de Windows desde una perspectiva Blue Team, diferenciando cuentas humanas, cuentas de equipo y servicios, y correlacionando una acción controlada desde `LAB-CLIENT01` con el evento generado en `LAB-DC01`.

## 1. Eventos estudiados

```text
4624 → inicio de sesión exitoso
4625 → inicio de sesión fallido
```

El hecho de encontrar un `4624` no basta para interpretar la actividad. Es necesario revisar, entre otros campos:

```text
TargetUserName
TargetDomainName
LogonType
IpAddress
WorkstationName
AuthenticationPackageName
```

## 2. Extracción de campos desde XML

Se consultaron eventos `4624` mediante PowerShell y se transformó el XML de cada evento en una tabla legible.

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id      = 4624
} -MaxEvents 50 |
ForEach-Object {

    $xml = [xml]$_.ToXml()
    $d = @{}

    foreach ($item in $xml.Event.EventData.Data) {
        $nombre = $item.GetAttribute('Name')
        $d[$nombre] = $item.InnerText
    }

    [PSCustomObject]@{
        Hora      = $_.TimeCreated
        Usuario   = $d['TargetUserName']
        Dominio   = $d['TargetDomainName']
        LogonType = $d['LogonType']
        IP        = $d['IpAddress']
        Equipo    = $d['WorkstationName']
        Auth      = $d['AuthenticationPackageName']
    }
} |
Format-Table -AutoSize
```

### Lección de depuración

Durante la práctica una ruta XML fue escrita incorrectamente como:

```powershell
$xml.even.eventdata.data
```

cuando debía ser:

```powershell
$xml.Event.EventData.Data
```

El resultado fue que la hora aparecía, pero los campos extraídos desde `$d[...]` quedaban vacíos.

La lección fue:

> Si el evento existe pero los campos extraídos aparecen vacíos, revisar primero la estructura XML y el parseo antes de asumir que Windows no registró la información.

## 3. Logon Types observados

En `LAB-DC01` se observaron, entre otros:

```text
LogonType 3 → Network
LogonType 5 → Service
LogonType 7 → Unlock
```

### LogonType 3 — Network

Representa un inicio de sesión utilizado para acceder a un recurso a través de la red.

Se observaron múltiples eventos asociados a cuentas de equipo como:

```text
LAB-DC01$
LAB-CLIENT01$
```

Las cuentas terminadas en `$` corresponden normalmente a identidades de máquina en Active Directory.

### LogonType 5 — Service

Se observaron eventos como:

```text
Usuario: SYSTEM
Dominio: NT AUTHORITY
LogonType: 5
Auth: Negotiate
```

Esto representa actividad de servicios de Windows, no a una persona iniciando sesión manualmente.

### LogonType 7 — Unlock

También se observó:

```text
Usuario: Administrador
LogonType: 7
```

Este tipo representa el desbloqueo de una sesión existente.

## 4. Prueba controlada desde LAB-CLIENT01

Para generar una autenticación de red de forma consciente se utilizó:

```powershell
klist purge
```

seguido de:

```powershell
dir \\LAB-DC01\SYSVOL
```

La acción obliga al cliente a autenticarse para acceder al recurso SMB/CIFS del controlador de dominio.

## 5. Evidencia obtenida en LAB-DC01

Al filtrar eventos `4624` asociados a `adm-mxnx`, se obtuvo de forma repetible:

```text
Usuario   : adm-mxnx
Dominio   : BLUELAB.TEST
LogonType : 3
IP        : 172.31.250.20
Auth      : Kerberos
```

La dirección `172.31.250.20` corresponde a `LAB-CLIENT01`.

Por lo tanto, la conclusión defendible es:

> Desde `LAB-CLIENT01` (`172.31.250.20`), la cuenta `adm-mxnx` accedió a un recurso de red del controlador de dominio. `LAB-DC01` registró un evento `4624` con `LogonType 3`, autenticado mediante Kerberos.

## 6. Correlación con la etapa Kerberos anterior

Esta evidencia se conecta con la práctica anterior:

```text
LAB-CLIENT01
→ adm-mxnx
→ acceso a \\LAB-DC01\SYSVOL
→ Kerberos
→ ticket CIFS visible con klist
→ eventos 4768/4769 en LAB-DC01
→ evento 4624 LogonType 3
→ IP origen 172.31.250.20
```

Así se correlacionan varias fuentes y capas de una misma acción:

```text
usuario
→ equipo origen
→ autenticación Kerberos
→ ticket de servicio
→ acceso de red
→ evento de logon exitoso en el DC
```

## 7. Lecciones Blue Team

- Un `4624` no se interpreta correctamente sin revisar `LogonType`.
- `LogonType 3` indica acceso por red, no un inicio interactivo frente al teclado.
- `LogonType 5` suele corresponder a servicios.
- `LogonType 7` corresponde a desbloqueo de sesión.
- Las cuentas terminadas en `$` son normalmente cuentas de equipo.
- La IP origen es clave para atribuir el evento al host que generó la actividad.
- `AuthenticationPackageName = Kerberos` permite relacionar el logon con la autenticación Kerberos ya observada.
- Generar actividad controlada y luego buscar el evento esperado es más útil que revisar logs sin hipótesis.
- La correlación temporal entre cliente y DC permite construir evidencia defendible.

## 8. Próximo paso

Generar de forma controlada un inicio de sesión fallido y estudiar el evento:

```text
4625 → inicio de sesión fallido
```

Campos a analizar:

```text
TargetUserName
LogonType
IpAddress
Status
SubStatus
FailureReason
AuthenticationPackageName
```

El objetivo será diferenciar no solo que una autenticación falló, sino **desde dónde, para qué usuario y por qué motivo**.