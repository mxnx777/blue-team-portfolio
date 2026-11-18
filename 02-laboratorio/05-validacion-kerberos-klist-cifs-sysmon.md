# Validación práctica de Kerberos con `klist`, CIFS/SMB, Sysmon y eventos 4768/4769

**Laboratorio:** `bluelab.test`  
**Cliente:** `LAB-CLIENT01` — `172.31.250.20`  
**Controlador de dominio / DNS / KDC:** `LAB-DC01` — `172.31.250.10`  
**Usuario observado:** `BLUELAB\adm-mxnx`

## Objetivo

Comprobar en un laboratorio real cómo una estación unida a Active Directory utiliza Kerberos contra el controlador de dominio y relacionar esa actividad con tickets Kerberos, Sysmon y eventos del registro Security del DC.

## 1. Detección inicial en Sysmon

Al analizar los `Event ID 3` de Sysmon y agrupar las conexiones por proceso y puerto se observó actividad de:

```text
lsass.exe → puerto 88
```

El puerto `88` corresponde a Kerberos.

Al filtrar por `lsass.exe` y puerto `88`, todas las conexiones observadas apuntaron a:

```text
172.31.250.10
```

La resolución DNS confirmó:

```text
172.31.250.10 → LAB-DC01
```

También `ipconfig /all` confirmó que `LAB-CLIENT01` pertenece a `bluelab.test` y utiliza `172.31.250.10` como servidor DNS.

## 2. Qué es el KDC

`KDC` significa **Key Distribution Center**.

En un dominio Windows, el servicio KDC reside en el controlador de dominio. En este laboratorio:

```text
KDC = LAB-DC01
```

Su función práctica es participar en la emisión de tickets Kerberos. La idea principal es que el usuario no envía su contraseña cada vez que necesita utilizar un servicio del dominio. Kerberos utiliza tickets temporales.

Flujo conceptual simplificado:

```text
adm-mxnx
→ se autentica en el dominio
→ el KDC valida la autenticación
→ entrega un TGT
→ el usuario necesita un servicio
→ presenta su TGT al KDC
→ obtiene un ticket específico para ese servicio
```

## 3. Qué muestra `klist`

Se ejecutó:

```powershell
klist
```

`klist` **no lista conexiones de red**. Muestra los tickets Kerberos almacenados en caché para la sesión actual.

Se observaron tickets asociados a `adm-mxnx`, entre ellos:

```text
krbtgt/BLUELAB.TEST
LDAP/LAB-DC01.bluelab.test
```

### TGT

`TGT` significa **Ticket Granting Ticket**.

El ticket:

```text
krbtgt/BLUELAB.TEST
```

es el ticket base que permite solicitar posteriormente tickets para servicios concretos del dominio.

### Cliente dentro de `klist`

Cuando aparece:

```text
Cliente: adm-mxnx @ BLUELAB.TEST
```

`Cliente` significa la identidad Kerberos propietaria del ticket. No significa necesariamente “computador cliente”.

## 4. Cuentas de usuario y cuentas de equipo

Active Directory mantiene identidades tanto para personas como para equipos.

Ejemplos del laboratorio:

```text
adm-mxnx       → cuenta de usuario
LAB-CLIENT01$  → cuenta de equipo
LAB-DC01$      → cuenta de equipo
```

El carácter `$` al final identifica normalmente una cuenta de equipo en Active Directory.

Esto explica por qué algunos eventos Kerberos pueden mostrar como identidad solicitante a `adm-mxnx` y otros a `LAB-CLIENT01$`: Windows también realiza autenticaciones en nombre del propio equipo.

## 5. Prueba controlada: purgar y volver a solicitar tickets

Se eliminó la caché Kerberos de la sesión:

```powershell
klist purge
```

Luego se provocó acceso a un recurso del dominio:

```powershell
dir \\LAB-DC01\SYSVOL
```

El acceso fue exitoso.

Después se ejecutó nuevamente:

```powershell
klist
```

Los tickets reaparecieron y se observó, entre otros:

```text
krbtgt/BLUELAB.TEST
cifs/LAB-DC01
```

El ticket `cifs/LAB-DC01` corresponde al servicio CIFS/SMB utilizado para acceder al recurso compartido `SYSVOL`.

## 6. Eventos Kerberos en el controlador de dominio

En `LAB-DC01` se consultó el registro `Security` y se observaron:

```text
4768 → solicitud de TGT Kerberos
4769 → solicitud de ticket de servicio Kerberos
```

Ejemplo de búsqueda:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id      = 4768,4769
} -MaxEvents 100 |
Where-Object {
    $_.Message -match 'adm-mxnx'
} |
Select-Object TimeCreated, Id, Message
```

Esto permitió filtrar la actividad Kerberos asociada específicamente al usuario observado.

## 7. Lectura de campos XML reales de un 4769

Se revisó directamente la vista XML de eventos `4769`.

En uno de los eventos asociados a `adm-mxnx` se observó:

```text
TargetUserName = adm-mxnx@BLUELAB.TEST
TargetDomainName = BLUELAB.TEST
ServiceName = LAB-DC01$
TicketEncryptionType = 0x12
IpAddress = ::ffff:172.31.250.20
```

Interpretación:

```text
adm-mxnx@BLUELAB.TEST
→ identidad que solicitó el ticket

LAB-DC01$
→ cuenta de equipo asociada al servicio de destino

::ffff:172.31.250.20
→ dirección IPv4 172.31.250.20 representada en formato IPv4-mapeado-en-IPv6
→ corresponde a LAB-CLIENT01

0x12
→ AES-256
```

También se observaron eventos cuyo `TargetUserName` era:

```text
LAB-CLIENT01$@BLUELAB.TEST
```

Esto evidencia solicitudes Kerberos realizadas usando la identidad de máquina del cliente.

## 8. Correlación temporal limpia

Para evitar depender de eventos antiguos se realizó una prueba nueva:

En `LAB-CLIENT01`:

```powershell
klist purge
dir \\LAB-DC01\SYSVOL
klist
```

`klist` mostró nuevamente:

```text
krbtgt/BLUELAB.TEST
cifs/LAB-DC01
KDC llamado: LAB-DC01.bluelab.test
```

Inmediatamente después, en `LAB-DC01`, se buscaron únicamente eventos recientes:

```powershell
$inicio = (Get-Date).AddMinutes(-5)

Get-WinEvent -FilterHashtable @{
    LogName   = 'Security'
    Id        = 4769
    StartTime = $inicio
} |
Where-Object {
    $_.Message -match 'adm-mxnx'
} |
Select-Object TimeCreated, Id, Message
```

Se observaron eventos `4769` nuevos dentro de la misma ventana temporal de la prueba.

Conclusión defendible:

> `adm-mxnx` en `LAB-CLIENT01` solicitó tickets Kerberos al controlador de dominio `LAB-DC01`; al acceder a `\\LAB-DC01\SYSVOL` se obtuvo un ticket de servicio `cifs/LAB-DC01` visible mediante `klist`, y el DC registró solicitudes de tickets de servicio mediante eventos `4769` en la misma ventana temporal.

## 9. Correlación conceptual completa

```text
adm-mxnx
→ trabaja desde LAB-CLIENT01
→ necesita acceder a \\LAB-DC01\SYSVOL
→ Kerberos utiliza el KDC de LAB-DC01
→ se dispone/obtiene un TGT
→ se solicita un ticket para el servicio
→ aparece cifs/LAB-DC01 en klist
→ se accede mediante SMB/CIFS
→ el DC registra actividad Kerberos 4768/4769
→ Sysmon permite observar actividad de red asociada
```

Esto conecta varias capas de una misma acción:

```text
identidad
→ equipo
→ proceso
→ protocolo
→ puerto
→ host destino
→ ticket Kerberos
→ evento del endpoint
→ evento del controlador de dominio
```

## 10. Puertos observados y significado

| Puerto | Servicio | Uso en este laboratorio |
|---:|---|---|
| 88 | Kerberos | autenticación y solicitud de tickets contra el KDC |
| 389 | LDAP | consultas/servicios de Active Directory |
| 445 | SMB/CIFS | acceso a recursos como SYSVOL |
| 135 | RPC Endpoint Mapper | descubrimiento y negociación de servicios RPC |
| puertos altos dinámicos | RPC dinámico | comunicaciones posteriores de RPC |

Estos puertos no se excluyeron del monitoreo solamente por generar eventos: tienen valor para entender y detectar actividad de dominio.

## 11. Lecciones Blue Team

- Un `Event ID 3` aislado no basta: proceso, IP, puerto y contexto deben correlacionarse.
- Antes de filtrar ruido hay que entender qué actividad representa.
- `lsass.exe → DC:88` es coherente con autenticación Kerberos en un cliente de dominio.
- `klist` muestra tickets Kerberos en caché, no conexiones de red.
- El KDC del laboratorio es `LAB-DC01`.
- El TGT permite solicitar tickets para servicios específicos.
- `adm-mxnx` es una cuenta de usuario; `LAB-CLIENT01$` y `LAB-DC01$` son cuentas de equipo.
- Los eventos `4768` y `4769` permiten observar solicitudes Kerberos desde el lado del controlador de dominio.
- Revisar XML permite identificar los nombres reales de los campos y evitar asumir estructuras.
- Filtrar por usuario y ventana temporal reduce ruido y permite una correlación más defendible.
- Una prueba controlada permite pasar de una hipótesis a evidencia reproducible.

## 12. Resultado

Se comprobó de forma práctica que `LAB-CLIENT01` utiliza Kerberos contra `LAB-DC01`, que `LAB-DC01` actúa como KDC, que `klist` permite observar los tickets almacenados en la sesión y que el acceso a `SYSVOL` provoca la presencia de un ticket `cifs/LAB-DC01`.

También se confirmó que el controlador de dominio registra solicitudes Kerberos mediante eventos `4768` y `4769`, incluyendo identidad solicitante, servicio, dirección IP y tipo de cifrado.

La práctica sirve como base para etapas posteriores de análisis de autenticación y detecciones relacionadas con Kerberos, manteniendo siempre la regla de no atribuir actividad sin evidencia suficiente.