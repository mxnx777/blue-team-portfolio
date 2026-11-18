# Guía práctica — Auditoría avanzada y análisis de eventos Windows

**Fecha de esta etapa:** 2026-08-11  
**Actualización:** 2026-08-18  
**Laboratorio:** `bluelab.test`  
**Controlador de dominio:** `LAB-DC01`  
**Cliente Windows 11:** `LAB-CLIENT01`  
**Objetivo:** activar auditoría de seguridad mediante GPO, generar eventos controlados en el endpoint y aprender a interpretar y correlacionar eventos como lo haría un analista Blue Team/SOC.

---

## 1. Punto de partida

La etapa anterior dejó operativo:

```text
BLUELAB\mxmx       → usuario estándar
BLUELAB\adm-mxnx   → cuenta administrativa separada
BLUELAB\GG-WKS-ADMINS
        ↓
GPO-WKS-ADMINISTRADORES-LOCALES
        ↓
LAB-CLIENT01\Administradores
```

La OU objetivo para estaciones de trabajo es:

```text
bluelab.test
└── BLUELAB
    └── EQUIPOS
        └── WORKSTATIONS
            └── LAB-CLIENT01
```

Concepto clave:

```text
OU = alcance lógico
GPO = configuración aplicada al alcance
Evento = evidencia generada en el endpoint
```

---

## 2. Creación de la GPO de auditoría

En `LAB-DC01` se abrió:

```text
Administrador del servidor
→ Herramientas
→ Administración de directivas de grupo
```

Ruta dentro de GPMC:

```text
Bosque: bluelab.test
└── Dominios
    └── bluelab.test
        └── BLUELAB
            └── EQUIPOS
                └── WORKSTATIONS
```

Sobre `WORKSTATIONS` se creó y vinculó:

```text
GPO-WKS-AUDITORIA-BLUE-TEAM
```

La GPO quedó vinculada a `WORKSTATIONS`, por lo que afecta a los equipos dentro de esa OU, actualmente `LAB-CLIENT01`, y no al controlador de dominio.

### Por qué se hizo así

Separar el alcance permite aplicar políticas de auditoría a estaciones de trabajo sin afectar accidentalmente a servidores o al DC.

---

## 3. Auditoría de inicio de sesión

Se editó:

```text
GPO-WKS-AUDITORIA-BLUE-TEAM
```

Ruta:

```text
Configuración del equipo
→ Directivas
→ Configuración de Windows
→ Configuración de seguridad
→ Configuración de directiva de auditoría avanzada
→ Directivas de auditoría
→ Inicio/cierre de sesión
→ Auditar inicio de sesión
```

Configuración:

```text
☑ Configurar los siguientes eventos de auditoría
☑ Correcto
☑ Error
```

Esto permite registrar:

```text
4624 → inicio de sesión exitoso
4625 → inicio de sesión fallido
```

---

## 4. Verificación de que la GPO llegó al endpoint

En `LAB-CLIENT01` se ejecutó:

```powershell
gpupdate /force
```

En una ejecución apareció temporalmente:

```text
La actualización de la directiva de equipo no se completó en el tiempo esperado.
```

En vez de asumir una falla de GPO, se verificó la cadena completa.

### 4.1 Conectividad con el DC

```powershell
ping 172.31.250.10
```

Resultado:

```text
4 enviados
4 recibidos
0% pérdida
```

### 4.2 DNS configurado en el cliente

```powershell
ipconfig /all
```

Resultado relevante:

```text
Servidor DNS: 172.31.250.10
Sufijo DNS: bluelab.test
```

### 4.3 Resolución DNS del DC

```powershell
nslookup lab-dc01.bluelab.test
```

Resultado relevante:

```text
Nombre: lab-dc01.bluelab.test
Address: 172.31.250.10
```

### 4.4 Descubrimiento del controlador de dominio

```powershell
nltest /dsgetdc:bluelab.test
```

Resultado:

```text
DC: \\LAB-DC01.bluelab.test
Dirección: \\172.31.250.10
El comando se completó correctamente
```

### 4.5 Acceso a SYSVOL y NETLOGON

```powershell
Test-Path "\\lab-dc01.bluelab.test\SYSVOL"
Test-Path "\\lab-dc01.bluelab.test\NETLOGON"
```

Resultado:

```text
True
True
```

### 4.6 Confirmación de GPO efectiva

```powershell
gpresult /r
```

En `Configuración de equipo` aparecieron:

```text
GPO-WKS-ADMINISTRADORES-LOCALES
GPO-WKS-AUDITORIA-BLUE-TEAM
Default Domain Policy
```

### Lección aprendida

Un error puntual de `gpupdate` no demuestra por sí solo que una GPO no se esté aplicando.

La validación correcta debe revisar:

```text
Red
→ DNS
→ descubrimiento del DC
→ SYSVOL/NETLOGON
→ gpresult
```

---

## 5. Validación de política de auditoría en LAB-CLIENT01

Se utilizó:

```powershell
auditpol /get /category:*
```

Resultado relevante:

```text
Inicio/cierre de sesión
Inicio de sesión    Aciertos y errores
```

Esto confirmó que la configuración de auditoría estaba activa en el endpoint.

---

## 6. Generación controlada de evento 4625

Para generar evidencia se realizó una prueba controlada en `LAB-CLIENT01`:

```text
1. Cerrar sesión.
2. Intentar iniciar sesión con BLUELAB\mxmx usando una contraseña incorrecta.
3. Iniciar sesión nuevamente con la contraseña correcta.
```

En:

```text
eventvwr.msc
→ Registros de Windows
→ Seguridad
```

se filtraron:

```text
4624,4625
```

### Evento 4625 observado

Datos relevantes:

```text
Evento: 4625
Equipo: LAB-CLIENT01.bluelab.test
Motivo del error: Nombre de usuario desconocido o contraseña incorrecta
Estado: 0xC000006D
Subestado: 0xC000006A
Tipo de inicio de sesión: 2
Estación de trabajo: LAB-CLIENT01
Dirección de red de origen: 127.0.0.1
Proceso de inicio de sesión: User32
Paquete de autenticación: Negotiate
```

### Interpretación

```text
4625                  → autenticación fallida
0xC000006A             → contraseña incorrecta
Tipo de inicio 2       → inicio de sesión interactivo
127.0.0.1              → origen local, el mismo endpoint
LAB-CLIENT01           → equipo donde ocurrió el intento
```

Conclusión del caso:

> Se detectó un intento local e interactivo de inicio de sesión fallido en `LAB-CLIENT01`, provocado deliberadamente en el laboratorio mediante una contraseña incorrecta.

### Mentalidad SOC

Un `4625` aislado no equivale automáticamente a un ataque.

Debe revisarse contexto:

```text
1 fallo aislado                         → posible error humano
muchos fallos al mismo usuario         → posible fuerza bruta
muchos usuarios desde un mismo origen  → posible password spraying
```

---

## 7. Auditoría de inicio de sesión especial

Dentro de la misma GPO se configuró:

```text
Configuración del equipo
→ Directivas
→ Configuración de Windows
→ Configuración de seguridad
→ Configuración de directiva de auditoría avanzada
→ Directivas de auditoría
→ Inicio/cierre de sesión
→ Auditar inicio de sesión especial
```

Configuración:

```text
☑ Configurar los siguientes eventos de auditoría
☑ Correcto
```

Esto permite registrar:

```text
4672 → privilegios especiales asignados a un nuevo inicio de sesión
```

---

## 8. Generación y análisis de evento 4672

Se inició sesión en `LAB-CLIENT01` con:

```text
BLUELAB\adm-mxnx
```

En el Visor de eventos se filtró:

```text
4672
```

Se observaron numerosos eventos, muchos correspondientes a cuentas y procesos legítimos del sistema como `SYSTEM`.

### Concepto importante

`4672` no significa:

```text
"un atacante obtuvo privilegios"
```

Significa:

```text
"a una nueva sesión se le asignaron privilegios especiales"
```

Windows genera eventos 4672 legítimos constantemente para cuentas y servicios privilegiados.

Por eso el análisis debe preguntar:

```text
¿qué cuenta recibió privilegios?
¿en qué equipo?
¿en qué momento?
¿qué tipo de inicio de sesión originó la sesión?
¿qué otros eventos pertenecen a la misma sesión?
```

### 4672 correspondiente a adm-mxnx

Se identificó manualmente un evento con:

```text
Nombre de cuenta: adm-mxnx
Dominio de cuenta: BLUELAB
Equipo: LAB-CLIENT01.bluelab.test
Id. de inicio de sesión: 0xF1EDDA
```

Privilegios observados incluían, entre otros:

```text
SeSecurityPrivilege
SeTakeOwnershipPrivilege
SeLoadDriverPrivilege
SeBackupPrivilege
SeRestorePrivilege
SeDebugPrivilege
SeSystemEnvironmentPrivilege
SeImpersonatePrivilege
SeDelegateSessionUserImpersonatePrivilege
```

---

## 9. Correlación manual 4624 ↔ 4672

Se tomó como pivote:

```text
Id. de inicio de sesión: 0xF1EDDA
```

Después se buscó manualmente un evento `4624` con el mismo identificador.

El `4624` encontrado correspondía a:

```text
Nombre de cuenta: adm-mxnx
Dominio de cuenta: BLUELAB
Tipo de inicio de sesión: 2
Proceso de inicio de sesión: User32
Id. de inicio de sesión: 0xF1EDDA
```

La correlación quedó:

```text
4624
→ BLUELAB\adm-mxnx inició sesión correctamente
→ Tipo 2: interactivo
→ Logon ID: 0xF1EDDA

4672
→ esa misma sesión recibió privilegios especiales
→ Logon ID: 0xF1EDDA
```

### Por qué el Logon ID es importante

El nombre del usuario no basta para correlacionar eventos porque un mismo usuario puede tener múltiples sesiones.

El `Logon ID` permite unir eventos pertenecientes a la misma sesión concreta.

Flujo de análisis:

```text
4672
↓
identificar cuenta
↓
obtener Logon ID
↓
buscar 4624 con el mismo Logon ID
↓
revisar tipo de inicio de sesión y origen
↓
interpretar el contexto
```

---

## 10. Auditoría de creación de procesos — Evento 4688

La siguiente etapa fue activar el registro de creación de procesos en estaciones de trabajo.

En `LAB-DC01` se editó:

```text
GPO-WKS-AUDITORIA-BLUE-TEAM
```

Ruta:

```text
Configuración del equipo
→ Directivas
→ Configuración de Windows
→ Configuración de seguridad
→ Configuración de directiva de auditoría avanzada
→ Directivas de auditoría
→ Seguimiento detallado
→ Auditar creación de procesos
```

Configuración aplicada:

```text
☑ Configurar los siguientes eventos de auditoría
☑ Correcto
```

Esto habilitó:

```text
4688 → Se creó un nuevo proceso
```

### Por qué sirve el 4688

El evento permite responder preguntas como:

```text
¿qué proceso se ejecutó?
¿quién lo ejecutó?
¿qué proceso padre lo lanzó?
¿con qué token o nivel de privilegio?
¿en qué equipo ocurrió?
¿en qué momento?
```

Esto es especialmente útil para investigar ejecución de herramientas como:

```text
cmd.exe
powershell.exe
whoami.exe
rundll32.exe
regsvr32.exe
mshta.exe
```

La presencia de uno de estos procesos no implica por sí sola actividad maliciosa; siempre se requiere contexto.

---

## 11. Primera observación de eventos 4688

Después de aplicar la GPO en `LAB-CLIENT01`, se observaron múltiples eventos 4688.

Ejemplos legítimos del sistema:

```text
svchost.exe
conhost.exe
smss.exe
msedgewebview2.exe
```

También se observó un proceso administrativo:

```text
Proceso nuevo: C:\Windows\System32\mmc.exe
Cuenta: BLUELAB\adm-mxnx
Token: TokenElevationTypeFull (2)
Nivel obligatorio: alto
```

### Lección aprendida

Un `4688` significa creación de proceso, no ataque.

La secuencia de análisis correcta es:

```text
4688
↓
usuario
↓
proceso nuevo
↓
proceso creador / padre
↓
token o nivel de integridad
↓
contexto
```

---

## 12. Habilitación de línea de comandos en 4688

Por defecto, conocer el nombre del ejecutable no siempre es suficiente.

No es lo mismo observar:

```text
powershell.exe
```

que observar:

```text
powershell.exe -Command Get-Date
```

Por eso se habilitó, dentro de la misma GPO:

```text
Configuración del equipo
→ Directivas
→ Plantillas administrativas
→ Sistema
→ Auditoría de creación de procesos
→ Incluir línea de comandos en eventos de creación de procesos
```

Estado:

```text
Habilitada
```

### Por qué es importante

Los argumentos muestran qué se pidió realmente al ejecutable.

Ejemplos conceptuales:

```text
powershell.exe -Command Get-Date
→ comportamiento simple y esperado en la prueba

powershell.exe -EncodedCommand ...
→ merece investigación y contexto adicional
```

La línea de comandos mejora notablemente la capacidad de investigación, pero también puede registrar información sensible introducida como argumento. En producción debe evaluarse este impacto antes de habilitarla masivamente.

---

## 13. Error de contexto detectado: mirar el equipo equivocado

Durante la prueba se revisaron inicialmente eventos 4688 del controlador de dominio.

En esos eventos aparecía:

```text
Equipo: LAB-DC01.bluelab.test
Línea de comandos de proceso: vacía
```

Esto no significaba que la nueva configuración hubiera fallado.

La GPO estaba vinculada a:

```text
BLUELAB\EQUIPOS\WORKSTATIONS
```

por lo tanto el equipo objetivo era:

```text
LAB-CLIENT01
```

Al revisar el registro correcto se confirmó la presencia de línea de comandos.

### Lección aprendida

Antes de interpretar un evento se debe comprobar siempre:

```text
¿qué equipo estoy mirando?
¿en qué OU está?
¿qué GPO debería recibir?
¿el evento corresponde al endpoint que estoy investigando?
```

Este error fue útil porque demuestra que una investigación puede desviarse aun cuando el Event ID sea correcto si se analiza el host equivocado.

---

## 14. Pruebas controladas con línea de comandos

Se ejecutaron en `LAB-CLIENT01` pruebas simples y benignas:

```powershell
cmd.exe /c whoami
powershell.exe -Command "Get-Date"
```

### 14.1 Caso whoami

Se identificó un evento 4688 con:

```text
Cuenta: BLUELAB\adm-mxnx
Proceso nuevo: C:\Windows\System32\whoami.exe
Proceso creador: C:\Windows\System32\cmd.exe
Línea de comandos: whoami
Equipo: LAB-CLIENT01.bluelab.test
```

La cadena observada fue:

```text
BLUELAB\adm-mxnx
        ↓
cmd.exe
        ↓
whoami.exe
```

Esto introduce el concepto de relación padre-hijo entre procesos.

### 14.2 Caso PowerShell

Se identificó otro evento 4688 con:

```text
Proceso nuevo:
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe

Línea de comandos:
"C:\WINDOWS\System32\WindowsPowerShell\v1.0\powershell.exe" -Command Get-Date

Equipo:
LAB-CLIENT01.bluelab.test
```

La diferencia clave es que ahora no solo sabemos que se abrió PowerShell, sino también qué instrucción recibió.

---

## 15. Qué campos revisar manualmente en un 4688

Para una investigación básica se debe leer al menos:

```text
Identificador de seguridad / cuenta
Nombre del nuevo proceso
Identificador del nuevo proceso
Nombre del proceso creador
Identificador del proceso creador
Línea de comandos
Tipo de elevación del token
Nivel obligatorio
Equipo
Fecha y hora
```

Método práctico:

```text
4688
↓
¿Quién?
↓
¿Qué ejecutó?
↓
¿Qué proceso lo lanzó?
↓
¿Qué argumentos llevaba?
↓
¿Con qué privilegio?
↓
¿Es esperable para ese usuario y equipo?
```

---

## 16. Eventos estudiados hasta esta etapa

| Event ID | Significado | Qué se validó |
|---|---|---|
| `4624` | Inicio de sesión exitoso | Inicio correcto y correlación con sesión privilegiada |
| `4625` | Inicio de sesión fallido | Contraseña incorrecta, tipo 2, origen local |
| `4672` | Privilegios especiales asignados | Sesión de `adm-mxnx` y diferenciación frente a eventos SYSTEM |
| `4688` | Creación de proceso | Proceso padre/hijo, cuenta, token y línea de comandos |

---

## 17. Forma de trabajo Blue Team aplicada

El objetivo no fue memorizar números de eventos, sino seguir un método repetible:

```text
1. Configurar telemetría
2. Verificar que realmente llegó al endpoint
3. Generar actividad controlada
4. Encontrar el evento
5. Leer campos relevantes
6. Interpretar el contexto
7. Correlacionar eventos relacionados
8. Diferenciar actividad normal de actividad que merece investigación
9. Documentar evidencia y conclusión
```

Ejemplos aprendidos:

```text
4625 por sí solo
≠ ataque confirmado

4672 por sí solo
≠ elevación maliciosa confirmada

4688 por sí solo
≠ ejecución maliciosa confirmada
```

El contexto decide si el comportamiento es normal o sospechoso.

---

## 18. Uso futuro en equipos sin dominio

La auditoría de Windows también puede configurarse en equipos que no pertenezcan a Active Directory mediante políticas locales, por ejemplo:

```text
secpol.msc
gpedit.msc
```

La diferencia operacional es:

```text
Con dominio
GPO central
→ múltiples equipos

Sin dominio
Política local
→ configuración por equipo o automatización posterior
```

Antes de aplicar configuraciones de laboratorio en producción se deben diseñar, probar y validar por separado.

---

## 19. Estado actual del laboratorio

En `LAB-CLIENT01` ya se dispone de telemetría nativa suficiente para comenzar a seguir actividad de autenticación y ejecución:

```text
4624 → inicio correcto
4625 → inicio fallido
4672 → privilegios especiales
4688 → creación de procesos
      + proceso padre
      + línea de comandos
      + nivel de elevación
```

La etapa de auditoría nativa de Windows queda funcional y validada para continuar con telemetría más rica.

---

## 20. Próximo paso

La siguiente etapa será **Sysmon**.

Objetivos previstos:

- instalar Sysmon de forma controlada en `LAB-CLIENT01`;
- comprender dónde registra sus eventos;
- observar creación de procesos con mayor riqueza de datos;
- revisar hashes, relaciones padre-hijo y conexiones de red según configuración;
- comparar telemetría nativa de Windows con Sysmon;
- preparar el envío posterior hacia Wazuh/SIEM;
- relacionar detecciones con MITRE ATT&CK.

Ruta formativa:

```text
4624 / 4625 / 4672 / 4688
        ↓
Sysmon
        ↓
Wazuh / SIEM
        ↓
MITRE ATT&CK
        ↓
casos documentados de detección e investigación
```
