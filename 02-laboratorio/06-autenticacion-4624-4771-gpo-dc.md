# Autenticación en Active Directory: 4624 LogonType 3, fallo Kerberos 4771 y auditoría por GPO

**Fecha:** 2026-08-26  
**Laboratorio:** `bluelab.test`  
**Cliente:** `LAB-CLIENT01` — `172.31.250.20`  
**Controlador de dominio:** `LAB-DC01` — `172.31.250.10`  
**Usuario de prueba:** `BLUELAB\adm-mxnx`

## Objetivo

Correlacionar autenticaciones exitosas y fallidas desde `LAB-CLIENT01` hacia `LAB-DC01`, identificar el tipo de inicio de sesión, comprobar el protocolo usado y verificar que la telemetría necesaria esté habilitada en el controlador de dominio.

La práctica se centró en:

```text
4624 → inicio de sesión exitoso
4771 → fallo de preautenticación Kerberos
LogonType 3 → inicio de sesión de red
```

---

## 1. Extracción estructurada de eventos 4624

El registro `Security` del DC contenía numerosos eventos `4624`.

Para evitar leer manualmente mensajes extensos, los eventos se convirtieron a XML y se extrajeron campos específicos:

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

### Lección de PowerShell

Una primera versión de la consulta devolvía solo la hora porque la ruta XML estaba mal escrita (`even` en lugar de `Event`).

Eso permitió aplicar una regla útil de troubleshooting:

> Si los campos existen en el XML pero la tabla sale vacía, primero se debe revisar la lógica de extracción antes de asumir que Windows no está registrando la información.

---

## 2. Tipos de inicio de sesión observados

En la tabla aparecieron distintos patrones legítimos:

```text
SYSTEM        → LogonType 5 → servicios
LAB-DC01$     → LogonType 3 → autenticación de red de cuenta de equipo
Administrador → LogonType 7 → desbloqueo de sesión
```

Interpretación práctica:

```text
LogonType 3  → Network
LogonType 5  → Service
LogonType 7  → Unlock
```

Esto demuestra que un evento `4624` no significa necesariamente que una persona haya escrito usuario y contraseña frente al equipo.

El contexto debe incluir:

```text
usuario
cuenta de equipo o usuario humano
LogonType
IP origen
paquete de autenticación
```

---

## 3. Prueba controlada de LogonType 3

Desde `LAB-CLIENT01` se realizó acceso a un recurso de red del DC:

```powershell
klist purge
dir \\LAB-DC01\SYSVOL
```

Luego, en `LAB-DC01`, se filtraron eventos `4624` asociados al usuario `adm-mxnx`.

Se obtuvo evidencia clara:

```text
Usuario   : adm-mxnx
Dominio   : BLUELAB.TEST
LogonType : 3
IP        : 172.31.250.20
Auth      : Kerberos
```

Interpretación:

> `adm-mxnx`, desde `LAB-CLIENT01` (`172.31.250.20`), accedió por red a un recurso del controlador de dominio. `LAB-DC01` registró un inicio de sesión exitoso `4624` de tipo `3` autenticado mediante Kerberos.

Esto correlaciona directamente:

```text
acción en cliente
→ acceso SMB/CIFS
→ autenticación Kerberos
→ evento 4624
→ LogonType 3
→ IP origen 172.31.250.20
```

---

## 4. Generación controlada de un fallo de autenticación

Primero se provocó un fallo local mediante:

```powershell
runas /user:BLUELAB\adm-mxnx cmd
```

usando una contraseña incorrecta.

Windows devolvió:

```text
ERROR RUNAS: no se puede ejecutar - cmd
1326: El nombre de usuario o la contraseña no son correctos.
```

Después se provocó un intento fallido contra un recurso de red del DC:

```powershell
net use \\LAB-DC01\IPC$ /user:BLUELAB\adm-mxnx 12345567
```

Resultado:

```text
Error de sistema 1326.
El nombre de usuario o la contraseña no son correctos.
```

---

## 5. Ausencia inicial de 4625 y 4771

En `LAB-DC01` se buscaron eventos recientes:

```text
4625 → inicio de sesión fallido
4771 → fallo de preautenticación Kerberos
```

No aparecieron eventos asociados a las pruebas.

En vez de seguir probando IDs al azar, se verificó la configuración real de auditoría del DC:

```powershell
auditpol /get /category:*
```

Se observó que varias subcategorías relevantes de Kerberos auditaban únicamente **Aciertos**.

Esto llevó a una conclusión operativa:

> Si no existe evidencia, primero se debe comprobar que la fuente esté configurada para generar esa evidencia.

---

## 6. GPO específica para controladores de dominio

La GPO existente:

```text
GPO-WKS-AUDITORIA-BLUE-TEAM
```

estaba vinculada a:

```text
BLUELAB\EQUIPOS\WORKSTATIONS
```

Por lo tanto no correspondía utilizarla para configurar auditoría específica del controlador de dominio.

Se creó una nueva GPO:

```text
GPO-DC-AUDITORIA-BLUE-TEAM
```

vinculada a:

```text
Domain Controllers
```

Dentro de:

```text
Configuración del equipo
→ Directivas
→ Configuración de Windows
→ Configuración de seguridad
→ Configuración de directiva de auditoría avanzada
→ Directivas de auditoría
→ Inicio de sesión de cuentas
```

se configuró:

```text
Auditar validación de credenciales
→ Aciertos y errores

Auditar servicio de autenticación Kerberos
→ Aciertos y errores

Auditar operaciones de vales de servicio Kerberos
→ Aciertos y errores
```

La política se aplicó y se verificó con `auditpol`.

Resultado observado:

```text
Operaciones de vales de servicio Kerberos → Aciertos y errores
Servicio de autenticación Kerberos         → Aciertos y errores
Validación de credenciales                 → Aciertos y errores
```

---

## 7. Repetición de la prueba y aparición del 4771

Después de aplicar correctamente la auditoría del DC se repitió un único intento de autenticación con contraseña incorrecta desde `LAB-CLIENT01`.

Esta vez apareció en `LAB-DC01`:

```text
Event ID 4771
Error de autenticación previa de Kerberos
```

El XML del evento mostró:

```text
TargetUserName = adm-mxnx
ServiceName    = krbtgt/BLUELAB
Status         = 0x18
PreAuthType    = 2
IpAddress      = ::ffff:172.31.250.20
IpPort         = 49808
```

Interpretación:

```text
adm-mxnx
→ usuario que intentó autenticarse

krbtgt/BLUELAB
→ solicitud relacionada con obtención de TGT

172.31.250.20
→ LAB-CLIENT01

::ffff:172.31.250.20
→ IPv4 representada como IPv4 mapeada en IPv6

Status 0x18
→ fallo de preautenticación Kerberos compatible con contraseña incorrecta

PreAuthType 2
→ preautenticación Kerberos basada en timestamp cifrado
```

---

## 8. Cadena completa de la investigación

### Autenticación correcta

```text
LAB-CLIENT01
172.31.250.20
→ adm-mxnx
→ acceso a \\LAB-DC01\SYSVOL
→ Kerberos
→ LAB-DC01
→ 4624
→ LogonType 3
→ Auth: Kerberos
```

### Autenticación fallida

```text
LAB-CLIENT01
172.31.250.20
→ adm-mxnx
→ contraseña incorrecta
→ intento de obtener TGT
→ LAB-DC01 / KDC
→ preautenticación rechazada
→ 4771
→ Status 0x18
```

---

## 9. Lecciones Blue Team

- Un `4624` necesita contexto; el `LogonType` cambia completamente su significado.
- `LogonType 3` representa autenticación de red.
- Las cuentas terminadas en `$` son cuentas de equipo y generan actividad legítima propia.
- `SYSTEM` con `LogonType 5` corresponde típicamente a servicios.
- Una autenticación fallida no implica automáticamente que el evento útil sea `4625`.
- Kerberos puede dejar evidencia específica como `4771`.
- Si un evento esperado no aparece, antes de asumir que no ocurrió se debe revisar la política de auditoría.
- `auditpol` permite conocer la configuración efectiva de auditoría.
- Las políticas del DC deben administrarse mediante una GPO aplicada a `Domain Controllers`, no reutilizando una GPO destinada a workstations.
- Generar una actividad controlada, predecir la evidencia esperada y después comprobarla es una metodología reproducible de análisis SOC.

---

## 10. Base para detección

Un único `4771` con `Status 0x18` puede ser simplemente un error humano.

Pero múltiples eventos similares en una ventana corta pueden justificar investigación.

Ejemplo conceptual:

```text
mismo usuario
+ múltiples 4771
+ Status 0x18
+ pocos minutos
→ posible password spraying / brute force / credenciales incorrectas repetidas
```

No se debe etiquetar automáticamente como ataque. La detección debe considerar frecuencia, origen, usuarios afectados y contexto.

---

## Resultado

La práctica logró correlacionar una autenticación exitosa de red mediante `4624 LogonType 3` y un fallo de preautenticación Kerberos mediante `4771`.

También permitió detectar y corregir una carencia de telemetría en el DC creando `GPO-DC-AUDITORIA-BLUE-TEAM` y habilitando auditoría de aciertos y errores para Kerberos y validación de credenciales.

Este bloque deja preparado el laboratorio para construir detecciones controladas basadas en intentos fallidos repetidos de autenticación.