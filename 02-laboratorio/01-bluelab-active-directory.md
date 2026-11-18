# Laboratorio Blue Team 01 — Active Directory, DNS y cliente Windows

**Estado:** En progreso  
**Fecha de inicio:** 2026-07-29  
**Tipo:** Laboratorio personal aislado  
**Objetivo profesional:** Practicar administración Windows, Active Directory, DNS, GPO, auditoría y detección Blue Team sin afectar redes productivas.

---

## 1. Objetivo técnico

Construir un entorno Windows corporativo pequeño y reproducible compuesto por:

- un controlador de dominio Windows Server 2022;
- un cliente Windows 11 Pro;
- una red virtual completamente aislada;
- Active Directory Domain Services y DNS interno;
- snapshots para repetir ejercicios y documentar evidencia.

Este laboratorio servirá posteriormente para practicar:

- creación y administración de usuarios, grupos y unidades organizativas;
- aplicación de GPO;
- auditoría de inicios de sesión y privilegios;
- Windows Event Logs y Sysmon;
- detección y respuesta ante incidentes;
- simulaciones controladas desde una VM atacante.

---

## 2. Equipo anfitrión

| Componente | Especificación |
|---|---|
| Sistema operativo | Windows 11 Pro |
| Hipervisor | VMware Workstation Pro 25H2 |
| Procesador | Intel Core i3-9100, 4 núcleos / 4 hilos |
| Memoria RAM | 16 GB |
| Almacenamiento | SSD 1 TB |

> La GPU no es relevante para este laboratorio. El recurso limitante principal es la RAM, por lo que las VMs se ejecutan con asignaciones controladas.

---

## 3. Diseño de red aislada

### Red virtual VMware

| Parámetro | Valor |
|---|---|
| Nombre | `LAB_ISOLADO` |
| VMnet | `VMnet2` |
| Tipo | Host-only personalizado |
| Subred | `172.31.250.0/24` |
| Adaptador virtual del host | Deshabilitado |
| DHCP VMware | Deshabilitado |
| NAT | No |
| Bridged | No |
| Puerta de enlace | No configurada |

### Regla de seguridad

Las máquinas virtuales del laboratorio solo pueden comunicarse entre ellas. No poseen puerta de enlace, no utilizan NAT y no están conectadas en modo puente a la red física.

```text
Internet / red corporativa
          X
          |
       VMnet2
   172.31.250.0/24
      LAB_ISOLADO
       /       \
LAB-DC01     LAB-CLIENT01
.10              .20
```

---

## 4. Máquinas virtuales

### LAB-DC01

| Parámetro | Valor |
|---|---|
| Sistema operativo | Windows Server 2022 Standard Desktop Experience |
| Función | Controlador de dominio y servidor DNS |
| CPU | 1 socket, 2 núcleos virtuales |
| RAM | 3 GB |
| Disco | 80 GB NVMe dinámico, archivo único |
| Firmware | UEFI |
| Red | Solo `LAB_ISOLADO` |
| IPv4 | `172.31.250.10/24` |
| Gateway | Vacío |
| DNS | `172.31.250.10` |
| Nombre del equipo | `LAB-DC01` |

### LAB-CLIENT01

| Parámetro | Valor |
|---|---|
| Sistema operativo | Windows 11 Pro 25H2 |
| Función | Cliente del dominio / equipo víctima controlado |
| CPU | 1 socket, 2 núcleos virtuales |
| RAM | 4 GB |
| Disco | 80 GB NVMe dinámico, archivo único |
| Firmware | UEFI + Secure Boot + TPM virtual |
| Red | Solo `LAB_ISOLADO` |
| IPv4 | `172.31.250.20/24` |
| Gateway | Vacío |
| DNS | `172.31.250.10` |
| Nombre del equipo | `LAB-CLIENT01` |
| Dominio | `bluelab.test` |

---

## 5. Dominio Active Directory

| Parámetro | Valor |
|---|---|
| Dominio DNS | `bluelab.test` |
| NetBIOS | `BLUELAB` |
| Nivel funcional del bosque | Windows Server 2016 |
| Nivel funcional del dominio | Windows Server 2016 |
| Controlador de dominio | `LAB-DC01.bluelab.test` |
| Catálogo global | Sí |
| DNS integrado en AD | Sí |
| RODC | No |

### Roles instalados

- Active Directory Domain Services.
- DNS Server.
- Herramientas de administración de AD DS.
- Group Policy Management.

---

## 6. Validaciones realizadas en LAB-DC01

### Dominio

```powershell
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, DomainMode
```

Resultado validado:

```text
DNSRoot     : bluelab.test
NetBIOSName : BLUELAB
DomainMode  : Windows2016Domain
```

### Dirección IP del controlador

```powershell
Get-NetIPAddress -AddressFamily IPv4 |
Where-Object {
    $_.IPAddress -ne "127.0.0.1" -and
    $_.IPAddress -notlike "169.254*"
} |
Format-Table InterfaceAlias, IPAddress, PrefixLength
```

Resultado validado:

```text
Ethernet0  172.31.250.10  24
```

### Servicios esenciales

```powershell
Get-Service DNS,NTDS,Netlogon,KDC |
Format-Table Name,Status,StartType
```

Resultado validado:

```text
DNS       Running  Automatic
KDC       Running  Automatic
Netlogon  Running  Automatic
NTDS      Running  Automatic
```

### Registro SRV de Active Directory

```powershell
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.bluelab.test
```

Resultado validado:

```text
NameTarget : lab-dc01.bluelab.test
Port       : 389
IPAddress  : 172.31.250.10
```

> En PowerShell el tipo correcto debe escribirse como `-Type SRV`. Escribir el nombre del registro completo dentro del parámetro `-Type` genera un error de enumeración y no representa una falla de DNS.

---

## 7. Incorporación de LAB-CLIENT01 al dominio

El cliente se incorporó por interfaz gráfica para comprender visualmente el proceso:

```text
sysdm.cpl
→ Nombre de equipo
→ Cambiar
→ Dominio: bluelab.test
→ Credencial autorizada: BLUELAB\Administrador
```

Resultado mostrado por Windows:

```text
Se unió correctamente al dominio BLUELAB.TEST.
```

### Diferencia entre credenciales de unión y sesión

Las credenciales `BLUELAB\Administrador` utilizadas durante la unión solamente autorizaron la operación. La sesión local que ya estaba abierta no cambió automáticamente.

Se cerró la sesión local y se inició una nueva sesión de dominio con:

```text
BLUELAB\Administrador
```

### Validación de identidad y equipo

```powershell
whoami
hostname
```

Resultado validado:

```text
bluelab\administrador
LAB-CLIENT01
```

Interpretación:

- `bluelab\administrador`: la identidad pertenece al dominio y es validada por Active Directory.
- `LAB-CLIENT01`: la sesión se ejecuta físicamente en el cliente Windows 11.

### Validación de pertenencia al dominio

```powershell
Get-CimInstance Win32_ComputerSystem |
Select-Object Name, Domain, PartOfDomain
```

Resultado validado:

```text
Name         : LAB-CLIENT01
Domain       : bluelab.test
PartOfDomain : True
```

> Error de aprendizaje: escribir `Select Object` en lugar de `Select-Object` provocó `PositionalParameterNotFound`. El guion forma parte del nombre oficial del cmdlet. Al corregirlo, la consulta funcionó correctamente.

---

## 8. Conceptos aprendidos

### Host, guest e identidad

| Concepto | Ejemplo en el laboratorio |
|---|---|
| Host físico | Windows 11 donde se ejecuta VMware |
| Guest / VM servidor | `LAB-DC01` |
| Guest / VM cliente | `LAB-CLIENT01` |
| Cuenta local | `LAB-CLIENT01\lab-client01` |
| Cuenta de dominio | `BLUELAB\Administrador` |

Comandos rápidos de orientación:

```powershell
hostname   # ¿En qué computador estoy?
whoami     # ¿Con qué identidad estoy trabajando?
```

### DNS y Active Directory

`LAB-CLIENT01` utiliza `172.31.250.10` como DNS porque ese servidor conoce los registros internos de `bluelab.test`, incluidos los registros SRV usados para descubrir LDAP, Kerberos y el controlador de dominio.

Un DNS público como `8.8.8.8` no conoce esos registros internos.

### Cuenta administrativa integrada

En Windows Server instalado en español, la cuenta integrada aparece como:

```text
BLUELAB\Administrador
```

La contraseña DSRM creada al promover el controlador de dominio no se usa para tareas administrativas normales. Se reserva para recuperación de Active Directory.

---

## 9. Snapshots

| VM | Snapshot | Estado |
|---|---|---|
| LAB-DC01 | `00-SRV2022-LIMPIO-SIN-ROLES` | Creado |
| LAB-DC01 | `01-DC01-AD-DNS-OPERATIVO` | Pendiente de confirmar |
| LAB-CLIENT01 | `00-W11-PRO-LIMPIO-IP` | Creado |
| LAB-CLIENT01 | `01-W11-UNIDO-A-BLUELAB` | Pendiente de crear |

Los snapshots se toman con las VMs apagadas para reducir tamaño y mantener estados consistentes.

---

## 10. Estado actual

| Control | Resultado |
|---|---|
| Red virtual aislada | Correcto |
| DC con IP estática | Correcto |
| AD DS instalado | Correcto |
| DNS instalado | Correcto |
| Dominio creado | Correcto |
| Servicios AD operativos | Correcto |
| Registros SRV DNS | Correcto |
| Cliente con IP y DNS interno | Correcto |
| Cliente unido al dominio | Correcto |
| Autenticación de dominio en el cliente | Correcto |
| Pertenencia `PartOfDomain=True` | Correcto |

---

## 11. Próximos pasos

### Fase inmediata

1. Apagar `LAB-CLIENT01`.
2. Crear snapshot `01-W11-UNIDO-A-BLUELAB`.
3. Confirmar el snapshot `01-DC01-AD-DNS-OPERATIVO`.
4. Crear una estructura profesional de unidades organizativas.
5. Crear un usuario normal para trabajo diario.
6. Crear una cuenta administrativa separada.
7. Evitar usar `Domain Admin` como cuenta cotidiana en el cliente.

### Fase de administración

- Crear estructura de OU.
- Crear usuarios y grupos de seguridad.
- Aplicar principio de mínimo privilegio.
- Crear GPO base para estaciones de trabajo.
- Configurar políticas de contraseñas y bloqueo.
- Crear cuentas administrativas separadas de las cuentas de uso diario.

### Fase Blue Team

- Activar auditoría avanzada.
- Instalar Sysmon.
- Centralizar eventos con Windows Event Forwarding.
- Analizar eventos 4624, 4625, 4672, 4688, 4720, 4728 y 4732.
- Simular acciones controladas y generar una línea de tiempo.
- Crear una VM Kali separada, conectada únicamente a `LAB_ISOLADO`.

---

## 12. Evidencia pendiente

```text
evidencia/
└── lab-01-active-directory/
    ├── 01-red-vmware/
    ├── 02-configuracion-dc/
    ├── 03-validacion-dns/
    ├── 04-union-cliente/
    ├── 05-ou-usuarios-grupos/
    ├── 06-gpo/
    └── 07-eventos-y-deteccion/
```

No subir:

- contraseñas;
- claves DSRM;
- códigos de licencia;
- información real de la red corporativa;
- nombres, RUT o datos personales reales;
- capturas que expongan credenciales.

---

## 13. Relación con NIST Cybersecurity Framework

| Función NIST | Aplicación en el laboratorio |
|---|---|
| Identify | Inventario de host, VMs, roles, red e IPs |
| Protect | Segmentación virtual, aislamiento, AD, DNS y mínimo privilegio |
| Detect | Event Logs, Sysmon y validación de eventos |
| Respond | Contención y procedimientos de respuesta controlados |
| Recover | Snapshots, restauración y repetición de escenarios |

---

## 14. Resultado profesional esperado

> Implementación de un dominio Active Directory y DNS en VMware, con direccionamiento estático, red virtual aislada, validación de servicios críticos, incorporación de clientes, autenticación centralizada, aplicación de GPO y generación de evidencia para prácticas Blue Team.
