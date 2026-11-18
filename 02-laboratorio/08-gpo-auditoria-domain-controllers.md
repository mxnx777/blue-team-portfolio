# Guía práctica — Crear una GPO de auditoría para Domain Controllers (GUI + comandos)

**Laboratorio:** `bluelab.test`  
**Controlador de dominio:** `LAB-DC01`  
**Cliente:** `LAB-CLIENT01`  
**Objetivo:** dejar documentado cómo crear, vincular, configurar, aplicar y verificar una GPO de auditoría específica para controladores de dominio, entendiendo por qué debe ir en `Domain Controllers` y no en otra OU.

---

## 1. Idea principal

Una GPO no se aplica “porque existe”. Se aplica porque está **vinculada** a un sitio, dominio u OU donde existen los objetos (usuarios/equipos) a los que quieres afectar.

En este laboratorio:

```text
bluelab.test
├── BLUELAB
│   └── EQUIPOS
│       └── WORKSTATIONS
│           └── LAB-CLIENT01
└── Domain Controllers
    └── LAB-DC01
```

Por eso:

```text
GPO-WKS-AUDITORIA-BLUE-TEAM
→ vinculada a WORKSTATIONS
→ afecta a LAB-CLIENT01

GPO-DC-AUDITORIA-BLUE-TEAM
→ vinculada a Domain Controllers
→ afecta a LAB-DC01
```

No se debe esperar que una GPO enlazada a `WORKSTATIONS` configure al DC, porque el objeto de equipo `LAB-DC01` no está dentro de esa OU.

---

## 2. Por qué la GPO de auditoría del DC va en `Domain Controllers`

Los controladores de dominio tienen un rol distinto a una estación de trabajo.

Un DC participa en:

- autenticación de usuarios y equipos;
- Kerberos / KDC;
- emisión de TGT y tickets de servicio;
- validación de credenciales;
- LDAP / Active Directory;
- políticas de dominio;
- registro de eventos críticos de autenticación.

Por eso interesa aplicar auditoría específica a los DC sin mezclarla con la telemetría de estaciones de trabajo.

Ejemplo:

```text
WORKSTATIONS
→ Sysmon
→ procesos
→ conexiones
→ auditoría de endpoint

Domain Controllers
→ Kerberos
→ Account Logon
→ Credential Validation
→ eventos 4768 / 4769 / 4771
→ autenticación central del dominio
```

Separar las GPO permite saber **qué política configura qué tipo de equipo**.

---

## 3. Por qué no modificar directamente Default Domain Controllers Policy

Existe una GPO predeterminada:

```text
Default Domain Controllers Policy
```

Pero para el laboratorio se creó una GPO separada:

```text
GPO-DC-AUDITORIA-BLUE-TEAM
```

Motivo:

- facilita saber qué cambios hicimos nosotros;
- evita mezclar configuración propia con políticas predeterminadas;
- simplifica troubleshooting;
- permite retirar o modificar la auditoría sin tocar la política base;
- mejora documentación y portafolio.

Regla práctica:

> Para configuraciones propias del laboratorio, preferir una GPO dedicada y bien nombrada antes que llenar las GPO predeterminadas con cambios difíciles de rastrear.

---

## 4. Orden mental antes de crear una GPO

Antes de tocar GPMC preguntarse:

```text
1. ¿Qué quiero configurar?
2. ¿Es configuración de usuario o de equipo?
3. ¿Qué objeto debe recibirla?
4. ¿En qué OU está ese objeto?
5. ¿Dónde debo vincular la GPO?
6. ¿Cómo voy a verificar que realmente se aplicó?
```

Ejemplo de esta práctica:

```text
¿Qué quiero configurar?
→ auditoría de autenticación Kerberos

¿Usuario o equipo?
→ equipo

¿Qué equipo?
→ LAB-DC01

¿Dónde está?
→ OU Domain Controllers

¿Dónde vinculo la GPO?
→ Domain Controllers

¿Cómo verifico?
→ gpresult + auditpol + evento 4771
```

---

# PARTE A — CREACIÓN POR GUI

## 5. Abrir Group Policy Management

En `LAB-DC01`:

```text
Administrador del servidor
→ Herramientas
→ Administración de directivas de grupo
```

También puede abrirse con:

```powershell
gpmc.msc
```

---

## 6. Crear la GPO y vincularla a Domain Controllers

Ruta:

```text
Bosque: bluelab.test
→ Dominios
→ bluelab.test
→ Domain Controllers
```

Click derecho sobre:

```text
Domain Controllers
```

Elegir:

```text
Crear un GPO en este dominio y vincularlo aquí...
```

Nombre:

```text
GPO-DC-AUDITORIA-BLUE-TEAM
```

Resultado esperado:

```text
Domain Controllers
├── Default Domain Controllers Policy
└── GPO-DC-AUDITORIA-BLUE-TEAM
```

---

## 7. Editar la GPO

Click derecho:

```text
GPO-DC-AUDITORIA-BLUE-TEAM
→ Editar
```

Ruta completa:

```text
Configuración del equipo
→ Directivas
→ Configuración de Windows
→ Configuración de seguridad
→ Configuración de directiva de auditoría avanzada
→ Directivas de auditoría
→ Inicio de sesión de cuentas
```

Esta parte es **Configuración del equipo** porque estamos configurando cómo debe auditar el controlador de dominio, no preferencias de un usuario.

---

## 8. Subcategorías configuradas

En el laboratorio se dejaron:

```text
Auditar validación de credenciales
→ Aciertos y errores

Auditar servicio de autenticación Kerberos
→ Aciertos y errores

Auditar operaciones de vales de servicio Kerberos
→ Aciertos y errores
```

La subcategoría:

```text
Auditar otros eventos de inicio de sesión de cuentas
```

quedó sin configurar en esta etapa porque no era necesaria para la prueba.

---

## 9. Qué controla cada subcategoría

### Servicio de autenticación Kerberos

Interesa para eventos relacionados con el KDC y TGT.

Ejemplo importante del laboratorio:

```text
4771 → Kerberos pre-authentication failed
```

Antes estaba solo en:

```text
Aciertos
```

Por eso la prueba de contraseña incorrecta no dejaba visible el `4771`.

Al pasar a:

```text
Aciertos y errores
```

se generó correctamente el evento.

### Operaciones de vales de servicio Kerberos

Permite observar operaciones de tickets de servicio.

Ejemplo estudiado:

```text
4769 → solicitud de ticket de servicio Kerberos
```

### Validación de credenciales

Es útil para observar validación de credenciales, especialmente en escenarios que pueden involucrar otros mecanismos como NTLM.

---

# PARTE B — ORDEN Y HERENCIA DE GPO

## 10. Orden general de procesamiento: LSDOU

Una forma clásica de recordar el orden de aplicación es:

```text
L → Local
S → Site
D → Domain
O → OU
```

Es decir:

```text
Local
→ Sitio
→ Dominio
→ OU padre
→ OU hija
```

Las políticas procesadas más tarde pueden sobrescribir configuraciones anteriores cuando existe conflicto, salvo comportamientos especiales como `Enforced` o `Block Inheritance`.

En este laboratorio no estamos usando esos mecanismos avanzados todavía.

---

## 11. Ejemplo práctico de jerarquía

Para `LAB-CLIENT01`:

```text
bluelab.test
└── BLUELAB
    └── EQUIPOS
        └── WORKSTATIONS
            └── LAB-CLIENT01
```

Una GPO vinculada directamente a `WORKSTATIONS` puede aplicarse a `LAB-CLIENT01`.

Para `LAB-DC01`:

```text
bluelab.test
└── Domain Controllers
    └── LAB-DC01
```

Una GPO vinculada a `WORKSTATIONS` no entra en la ruta de procesamiento del DC.

Por eso se creó una segunda GPO específica.

---

## 12. Si hay varias GPO en la misma OU

Si existen varias GPO vinculadas al mismo contenedor, GPMC mantiene un **Link Order**.

En términos simples:

```text
varias GPO en una OU
→ todas pueden aplicar
→ si dos configuran lo mismo de forma incompatible
→ el orden de vínculo puede influir en cuál prevalece
```

Para este laboratorio la mejor práctica es evitar conflictos innecesarios:

```text
una GPO con propósito claro
→ nombre claro
→ alcance claro
→ pocas configuraciones relacionadas
```

Ejemplo:

```text
GPO-WKS-ADMINISTRADORES-LOCALES
GPO-WKS-AUDITORIA-BLUE-TEAM
GPO-DC-AUDITORIA-BLUE-TEAM
```

---

# PARTE C — CREAR Y VINCULAR LA GPO POR COMANDOS

## 13. Importar el módulo GroupPolicy

En `LAB-DC01`, PowerShell como administrador:

```powershell
Import-Module GroupPolicy
```

Verificar que esté disponible:

```powershell
Get-Command -Module GroupPolicy
```

---

## 14. Crear la GPO por PowerShell

```powershell
New-GPO -Name 'GPO-DC-AUDITORIA-BLUE-TEAM'
```

Esto **crea el objeto GPO**, pero todavía no significa que se aplique a ningún equipo.

---

## 15. Vincular la GPO a Domain Controllers

Distinguished Name de la OU:

```text
OU=Domain Controllers,DC=bluelab,DC=test
```

Comando:

```powershell
New-GPLink `
    -Name 'GPO-DC-AUDITORIA-BLUE-TEAM' `
    -Target 'OU=Domain Controllers,DC=bluelab,DC=test' `
    -LinkEnabled Yes
```

Concepto:

```text
New-GPO
→ crea la política

New-GPLink
→ decide dónde se aplica
```

Crear sin vincular no basta.

---

## 16. Consultar la GPO por comando

```powershell
Get-GPO -Name 'GPO-DC-AUDITORIA-BLUE-TEAM'
```

Para obtener un informe:

```powershell
Get-GPOReport `
    -Name 'GPO-DC-AUDITORIA-BLUE-TEAM' `
    -ReportType Html `
    -Path 'C:\Temp\GPO-DC-AUDITORIA-BLUE-TEAM.html'
```

Esto sirve para revisar/documentar la configuración.

---

## 17. Importante: configurar Advanced Audit Policy por comandos

PowerShell nativo permite crear y vincular la GPO fácilmente, pero no ofrece un cmdlet simple equivalente a:

```text
Set-GPAuditPolicy -Subcategory ...
```

para editar estas subcategorías de Advanced Audit Policy dentro de una GPO.

Por claridad en este laboratorio:

```text
crear/vincular GPO
→ PowerShell o GUI

configurar Advanced Audit Policy dentro de la GPO
→ GUI (GPMC)

verificar resultado efectivo en el DC
→ auditpol / gpresult
```

No se debe confundir el siguiente comando:

```powershell
auditpol /set ...
```

con “configurar la GPO”. `auditpol /set` cambia la política efectiva/local del equipo donde se ejecuta y es útil para pruebas, pero no reemplaza la administración centralizada mediante GPO.

---

# PARTE D — COMANDOS DE PRUEBA Y VERIFICACIÓN

## 18. Forzar actualización de políticas

En `LAB-DC01`:

```powershell
gpupdate /force
```

Esto solicita que el equipo vuelva a procesar las políticas aplicables.

---

## 19. Verificar qué GPO recibió el DC

Resumen:

```powershell
gpresult /r /scope computer
```

Buscar:

```text
GPO-DC-AUDITORIA-BLUE-TEAM
```

Informe HTML más completo:

```powershell
gpresult /h C:\Temp\gpresult-LAB-DC01.html
```

Luego abrir:

```powershell
start C:\Temp\gpresult-LAB-DC01.html
```

---

## 20. Verificar auditoría efectiva con auditpol

Todas las categorías:

```powershell
auditpol /get /category:*
```

Solo Kerberos Authentication Service:

```powershell
auditpol /get /subcategory:'Servicio de autenticación Kerberos'
```

Credential Validation:

```powershell
auditpol /get /subcategory:'Validación de credenciales'
```

Resultado esperado:

```text
Aciertos y errores
```

---

## 21. Equivalente local por comandos para una prueba rápida

Solo para prueba/control local en el DC:

```powershell
auditpol /set /subcategory:'Servicio de autenticación Kerberos' /success:enable /failure:enable
```

```powershell
auditpol /set /subcategory:'Validación de credenciales' /success:enable /failure:enable
```

```powershell
auditpol /set /subcategory:'Operaciones de vales de servicio Kerberos' /success:enable /failure:enable
```

Después verificar:

```powershell
auditpol /get /category:*
```

Recordatorio:

> Esto es útil para comprobar comportamiento, pero la configuración definitiva del laboratorio debe quedar gobernada por `GPO-DC-AUDITORIA-BLUE-TEAM`.

---

# PARTE E — PRUEBA REAL QUE VALIDÓ LA GPO

## 22. Antes de habilitar auditoría de errores

Desde `LAB-CLIENT01` se provocó una contraseña incorrecta:

```powershell
net use \\LAB-DC01\IPC$ /user:BLUELAB\adm-mxnx ClaveMala123!
```

Resultado:

```text
Error de sistema 1326
El nombre de usuario o la contraseña no son correctos.
```

Pero en `LAB-DC01` no aparecía:

```text
4771
```

Se comprobó con:

```powershell
auditpol /get /category:*
```

que `Servicio de autenticación Kerberos` estaba configurado solo en:

```text
Aciertos
```

---

## 23. Después de aplicar GPO-DC-AUDITORIA-BLUE-TEAM

Se dejó:

```text
Servicio de autenticación Kerberos
→ Aciertos y errores
```

Se repitió una sola vez la prueba desde `LAB-CLIENT01`.

En `LAB-DC01` apareció:

```text
Event ID 4771
→ Error de autenticación previa de Kerberos
```

Campos observados:

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
→ intentó autenticarse

172.31.250.20
→ LAB-CLIENT01

krbtgt/BLUELAB
→ solicitud relacionada con TGT

0x18
→ preautenticación Kerberos fallida / contraseña incorrecta
```

---

# PARTE F — CHECKLIST OPERATIVO

## 24. Si mañana necesito crear otra GPO

Checklist:

```text
[ ] Definir qué quiero cambiar
[ ] Determinar si es usuario o equipo
[ ] Identificar el objeto objetivo
[ ] Identificar su OU
[ ] Crear GPO con nombre claro
[ ] Vincularla a la OU correcta
[ ] Configurar solo lo necesario
[ ] gpupdate /force
[ ] gpresult para comprobar aplicación
[ ] auditpol / comando específico para comprobar efecto
[ ] generar una prueba controlada
[ ] revisar el evento esperado
[ ] documentar resultado
```

---

## 25. Regla para no confundirse

```text
GPO = QUÉ quiero configurar

LINK = DÓNDE quiero aplicarlo

OU = A QUIÉN afecta según dónde está el objeto

GPRESULT = QUÉ GPO recibió realmente el equipo

AUDITPOL = QUÉ auditoría está efectiva realmente

EVENT VIEWER / Get-WinEvent = QUÉ evidencia se está generando realmente
```

Si una política “no funciona”, revisar en ese orden antes de cambiar cosas al azar.

---

## 26. Resumen final

La GPO `GPO-DC-AUDITORIA-BLUE-TEAM` se creó separada de la política de estaciones porque `LAB-DC01` pertenece a la OU `Domain Controllers` y tiene funciones de autenticación distintas de un endpoint.

El flujo correcto aprendido fue:

```text
identificar objetivo
→ localizar OU
→ crear GPO
→ vincularla a Domain Controllers
→ configurar Advanced Audit Policy
→ gpupdate
→ gpresult
→ auditpol
→ generar prueba controlada
→ confirmar evento 4771
```

La lección principal es que **crear una GPO, vincularla y comprobar su efecto son tres cosas distintas**. Nunca asumir que una política se aplica solo porque aparece en GPMC.
