# Cierre de etapa — Windows, Active Directory, Sysmon, Kerberos y plan de repetición

**Laboratorio:** `bluelab.test`  
**Controlador de dominio:** `LAB-DC01`  
**Cliente:** `LAB-CLIENT01`  
**Objetivo general:** consolidar una base Blue Team/SOC útil para empleabilidad antes de avanzar a Wazuh/SIEM.

---

## 1. Qué quedó aprendido y validado

Durante esta etapa se trabajó de forma práctica sobre:

- Active Directory y estructura de OU.
- GPO para estaciones de trabajo y para controladores de dominio.
- Auditoría avanzada de Windows.
- Eventos de seguridad `4624`, `4625`, `4768`, `4769` y `4771`.
- Sysmon Event ID `1`, `3` y `5`.
- Correlación entre procesos, conexiones de red, tickets Kerberos y eventos del DC.
- Uso de `klist`.
- Uso de `auditpol`, `gpupdate` y `gpresult`.
- Lectura de XML real de eventos.
- PowerShell práctico para traer, filtrar, agrupar, ordenar y resumir eventos.

La meta no es memorizar todos los comandos, sino poder explicar la lógica y reconstruir el flujo de investigación.

---

## 2. Metodología de trabajo a partir de ahora

El laboratorio seguirá tres capas:

### 2.1. Aprender

Primero se realiza una práctica guiada entendiendo:

```text
qué quiero observar
→ qué telemetría necesito
→ qué evento debería aparecer
→ cómo lo genero
→ cómo lo valido
→ qué significa
```

### 2.2. Repetir

Luego se repite el mismo bloque con menos ayuda.

La meta es que el procedimiento deje de depender de una receta completa y pase a depender de comprensión.

### 2.3. Demostrar

Finalmente se reconstruye o investiga el escenario desde cero.

El resultado debe ser defendible en una entrevista técnica:

```text
"sé montar esto"
"sé generar el evento"
"sé encontrarlo"
"sé correlacionarlo"
"sé explicar por qué ocurrió"
```

---

## 3. Qué NO se busca

Este laboratorio no busca:

- memorizar sintaxis de PowerShell como si fuera programación académica;
- crear dashboards por estética;
- acumular herramientas sin comprenderlas;
- copiar configuraciones grandes sin saber qué hacen;
- generar documentación de relleno.

Se prioriza lo que tenga valor para un cargo Blue Team/SOC.

---

## 4. PowerShell: objetivo real

El objetivo es reconocer patrones como:

```powershell
Get-WinEvent
Where-Object
Group-Object
Sort-Object
Select-Object
ForEach-Object
```

Y poder interpretar conceptualmente:

```text
Get-WinEvent
→ traer eventos

Where-Object
→ filtrar

Group-Object
→ agrupar

Sort-Object
→ ordenar

Select-Object
→ mostrar solo los campos relevantes

ForEach-Object
→ procesar cada evento
```

La sintaxis exacta puede consultarse y reutilizarse. La capacidad importante para SOC es saber qué información buscar y cómo estructurar la consulta.

---

## 5. Prácticas ya consolidadas

### 5.1. Sysmon y red

Se observó actividad de red con Event ID `3` y se correlacionó:

```text
lsass.exe
→ puerto 88
→ 172.31.250.10
→ LAB-DC01
→ Kerberos
```

También se trabajó con:

```text
389 → LDAP
445 → SMB/CIFS
135 → RPC Endpoint Mapper
```

### 5.2. Kerberos

Se validaron:

```text
4768 → solicitud de TGT
4769 → solicitud de ticket de servicio
4771 → fallo de preautenticación Kerberos
```

Se utilizó `klist` para observar:

```text
krbtgt/BLUELAB.TEST
cifs/LAB-DC01
```

### 5.3. Inicio de sesión

Se validó un `4624` con:

```text
Usuario   = adm-mxnx
LogonType = 3
IP        = 172.31.250.20
Auth      = Kerberos
```

Esto permitió asociar un acceso de red desde `LAB-CLIENT01` con un evento exitoso en `LAB-DC01`.

### 5.4. Fallo de contraseña Kerberos

Se generó una contraseña incorrecta desde `LAB-CLIENT01` y, después de corregir la auditoría del DC, apareció:

```text
Event ID        = 4771
TargetUserName  = adm-mxnx
Status          = 0x18
PreAuthType     = 2
IpAddress       = ::ffff:172.31.250.20
```

`0x18` fue interpretado como fallo de preautenticación asociado a contraseña incorrecta.

---

## 6. Lección crítica: telemetría antes que hipótesis

Cuando inicialmente no aparecía `4771`, no se debía concluir que la prueba estaba mal.

Se verificó:

```powershell
auditpol /get /category:*
```

Y se detectó que el servicio de autenticación Kerberos auditaba solo aciertos.

Se creó una GPO específica para controladores de dominio:

```text
GPO-DC-AUDITORIA-BLUE-TEAM
```

vinculada a:

```text
Domain Controllers
```

Después se habilitaron aciertos y errores para las subcategorías relevantes.

Resultado:

```text
prueba repetida
→ aparece 4771
```

Principio consolidado:

> Si un evento esperado no aparece, antes de cambiar de hipótesis hay que comprobar que la fuente esté configurada para generarlo.

---

## 7. Primera lógica de detección

Se generaron varios `4771` controlados con:

```text
mismo usuario
+ misma IP
+ mismo código 0x18
+ pocos segundos de diferencia
```

También aparecieron dos eventos reales de la cuenta `Administrador` con origen `::1` porque se escribió mal la contraseña localmente.

Esto permitió distinguir:

```text
detección
≠
ataque confirmado
```

El analista debe agregar contexto:

```text
quién
desde dónde
cuántas veces
en cuánto tiempo
si la actividad era esperada
si continúa aumentando
```

---

## 8. Snapshots antes de Wazuh

Antes de continuar a Wazuh/SIEM se recomienda crear snapshots de:

```text
LAB-DC01
LAB-CLIENT01
```

Nombre sugerido:

```text
BASE-AD-SYSMON-AUDITORIA-OK
```

Objetivo:

- conservar el laboratorio funcional;
- poder volver atrás si una instalación rompe algo;
- permitir una futura reconstrucción/repetición desde cero.

---

## 9. Plan de repetición futura

Antes de considerar esta base dominada, se repetirá de forma progresivamente más autónoma:

1. Crear una GPO y decidir dónde vincularla.
2. Verificar aplicación con `gpresult`.
3. Verificar auditoría efectiva con `auditpol`.
4. Generar un `4624` controlado.
5. Generar `4768`/`4769` con Kerberos.
6. Generar un `4771` controlado.
7. Filtrar y resumir eventos con PowerShell.
8. Explicar qué pasó sin depender de una receta completa.

Después se realizará una reconstrucción del laboratorio desde cero:

```text
Windows Server
→ AD DS
→ dominio
→ DNS
→ OUs
→ usuarios/grupos
→ cliente unido al dominio
→ GPO
→ auditoría
→ Sysmon
→ Wazuh
```

---

## 10. Siguiente etapa: Wazuh/SIEM

La arquitectura objetivo será:

```text
LAB-CLIENT01 ─┐
              ├─> Wazuh Manager / Indexer / Dashboard
LAB-DC01    ──┘
```

Objetivos de esta etapa:

- centralizar logs Windows y Sysmon;
- instalar y registrar agentes;
- visualizar eventos desde una consola SIEM;
- generar alertas;
- investigar alertas;
- crear reglas básicas;
- relacionar detecciones con MITRE ATT&CK;
- documentar casos de uso en GitHub.

La meta no será "tener Wazuh instalado", sino poder demostrar un flujo completo:

```text
actividad
→ telemetría
→ envío al SIEM
→ alerta
→ investigación
→ contexto
→ conclusión
→ documentación
```

---

## 11. Criterio de empleabilidad

A partir de esta etapa, cada práctica debe responder al menos una de estas preguntas:

- ¿me ayuda a operar un SIEM?
- ¿me ayuda a investigar una alerta?
- ¿me ayuda a entender Windows/AD/redes?
- ¿me ayuda a automatizar una tarea repetitiva?
- ¿me ayuda a explicar una investigación en entrevista?
- ¿produce evidencia útil para portafolio?

Si no aporta a esos objetivos, se deja fuera o se posterga.

---

## 12. Estado del laboratorio al cierre

```text
AD / dominio                 → operativo
LAB-DC01                     → operativo
LAB-CLIENT01                 → unido al dominio
GPO workstation              → operativa
GPO Domain Controllers       → operativa
Auditoría Security           → validada
Sysmon                       → operativo
Kerberos                     → observado y correlacionado
4624 / 4768 / 4769 / 4771    → validados
PowerShell para hunting      → nivel inicial práctico
Primera lógica de detección  → realizada
Próximo bloque               → Wazuh / SIEM
```

Este punto funciona como checkpoint técnico antes de comenzar la etapa SIEM.