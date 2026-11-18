# Acceso temporal de proveedor y simulación de incidente

**Tipo:** casos reales en entorno laboral (versión anonimizada)  
**Servidor:** `SRV-APP01`, Windows Server 2025, IIS + SQL Server, workgroup

---

## Parte A — Habilitación controlada de acceso a proveedor externo

### Contexto
Un proveedor necesitaba intervenir el servidor de aplicaciones para un paso a producción de un módulo web.

### Antes de la intervención: estado y respaldos
- Estado validado: instancia SQL Server en ejecución, sitio IIS iniciado, ruta física documentada.
- Respaldos:

```powershell
Backup-WebConfiguration -Name PREVIO_INTERVENCION_AAAA-MM-DD
```

  más una copia manual de `applicationHost.config`, `administration.config` y del contenido de `wwwroot`.

### Acceso entregado
- Cuenta local dedicada para el proveedor (no se compartió una cuenta existente).
- Login SQL asociado a esa cuenta.
- Vigencia operativa definida: 48 horas.
- Riesgos documentados al momento de habilitar: el sitio operaba solo por HTTP 80, sin binding HTTPS ni host header.

### Cierre definido (revocación)
- Quitar el rol SQL y eliminar el login.
- Deshabilitar o eliminar la cuenta local.
- Rotar credenciales compartidas si corresponde.

### Autocrítica: qué haría distinto hoy
El acceso se entregó como **administrador local + `sysadmin` en SQL**, más de lo que probablemente necesitaba la tarea. Con lo que he estudiado después, lo haría así:

| Control | Cómo |
|---|---|
| Mínimo privilegio | Rol SQL acotado a la base de datos del módulo, no `sysadmin` |
| Expiración automática | `net user <cuenta> /expires:<fecha>`, en vez de depender de revocarla a mano |
| Monitoreo durante la ventana | Vigilar eventos `4624` (logon), `4720` (cuenta creada) y `4732` (agregado a grupo local) |
| Evidencia de cierre | Captura de la cuenta deshabilitada y del login eliminado |

---

## Parte B — Simulación de incidente (tabletop + validación técnica)

**Fecha:** enero 2026  
**Escenario simulado:** *"Un equipo interno fue comprometido y se intenta mover lateralmente hacia el servidor de aplicaciones usando credenciales válidas."*  
**Regla del ejercicio:** no se ataca; solo se observan señales con comandos de lectura, sin afectar producción.

### Señal 1 — Conexiones establecidas

```powershell
Get-NetTCPConnection |
  Where-Object {$_.State -eq "Established"} |
  Select-Object LocalAddress,LocalPort,RemoteAddress,RemotePort
```

**Qué mostró:**
- Conexiones entrantes a SQL Server (1433) desde un grupo acotado de estaciones internas: es el patrón normal de la aplicación.
- Una conexión entrante al puerto 7070 desde otra estación. Ese puerto lo usa AnyDesk para conexiones directas, y AnyDesk estaba instalado como servicio (ver revisión posterior): es acceso remoto activo y hay que validar quién lo usaba.
- Conexiones salientes a servicios en la nube (443 y 8883, este último típico de MQTT/telemetría).

**Uso para detección:** con esta línea base, una conexión a 1433 desde una estación **fuera** de ese grupo pasa a ser una anomalía investigable.

### Señal 2 — Inicios de sesión administrativos

```powershell
Get-WinEvent -LogName Security -MaxEvents 50 |
  Where-Object {$_.Id -in 4624,4672} |
  Select TimeCreated,Id,Message
```

**Qué mostró:** decenas de pares **4624 + 4672 con la misma marca de tiempo** a lo largo de la mañana.

**Interpretación:** el 4672 ("se asignaron privilegios especiales") que acompaña a un 4624 indica que **ese inicio de sesión tiene privilegios administrativos**. En un movimiento lateral con credenciales válidas, esta es la huella principal. El siguiente paso de análisis sería extraer de cada 4624 la cuenta, el `LogonType` y la IP de origen, y separar los inicios de servicios del sistema de los de cuentas humanas.

### Señal 3 — Actividad de WinRM (PowerShell remoting)

```powershell
Get-WinEvent -LogName Microsoft-Windows-WinRM/Operational -MaxEvents 30 |
  Select TimeCreated,Message
```

**Qué mostró:** secuencias completas de creación de sesión WSMan → creación de shell → ejecución de comandos (`CommandId`) → cierre, con conexión a `localhost:47001`.

**Interpretación:** WinRM es una vía clásica de movimiento lateral y un log que pocas veces se revisa. Aquí la conexión era local (`localhost`), lo que es consistente con administración en el propio servidor. Si la sesión viniera de una IP remota no esperada, sería una señal fuerte.

### Contención diseñada (documentada, no ejecutada)

Principios: **no** desenchufar, **no** apagar, **no** borrar nada (se perdería evidencia volátil). Reducir la superficie de ataque temporalmente:

```powershell
# Bloqueo lógico de WinRM. Solo en simulación: en producción requiere planificación.
New-NetFirewallRule -DisplayName "SIMULACION_BLOQUEO_WINRM" `
  -Protocol TCP -LocalPort 5985 -Action Block
```

Además, inventario de servicios en ejecución (`Get-Service | Where-Object {$_.Status -eq "Running"}`) para distinguir cuáles son críticos (SQL Server, IIS) y cuáles se podrían detener temporalmente.

### Revisión posterior del inventario de servicios

Releyendo la salida con lo estudiado después:
- **Herramienta de acceso remoto instalada como servicio permanente:** si se usa solo para soporte puntual, conviene que no quede corriendo en forma continua, porque es una puerta de entrada conocida.
- **Cola de impresión (`Spooler`) activa en un servidor de aplicaciones:** si el servidor no imprime, se recomienda deshabilitarla (superficie de ataque conocida, por ejemplo PrintNightmare).

### Qué aprendí
- Sin baseline no hay forma de decir qué es anómalo.
- 4624 + 4672 juntos significan un inicio de sesión con privilegios: es lo primero que se mira ante una sospecha de movimiento lateral.
- Contener sin conservar evidencia destruye la investigación.
