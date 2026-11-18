# LAB-SOC-2026-003 — Fallo de autenticación Kerberos invisible: brecha de auditoría en el DC

**Fecha:** 2026-08-26  
**Entorno:** laboratorio BLUELAB, `LAB-CLIENT01` (`172.31.250.20`) → `LAB-DC01` (`172.31.250.10`)  
**Fuente:** Windows Security Log del controlador de dominio  
**Severidad asignada:** Media (brecha de visibilidad: un ataque de contraseñas no habría sido detectado)  
**Estado:** concluido, brecha corregida

## 1. Resumen

Se provocaron intentos de autenticación con contraseña incorrecta contra el DC. **No apareció ningún evento de fallo.** En vez de seguir probando, se revisó la configuración de auditoría: el DC solo auditaba aciertos de Kerberos. Se corrigió con una GPO dedicada a Domain Controllers y, al repetir la prueba, apareció el evento **4771** con todos los campos necesarios para investigar.

**Hallazgo principal:** con la configuración inicial, un password spraying contra el dominio habría pasado desapercibido.

## 2. Prueba controlada

```powershell
# Acceso exitoso (línea base)
klist purge
dir \\LAB-DC01\SYSVOL

# Fallos provocados con contraseña incorrecta
runas /user:BLUELAB\adm-mxnx cmd
net use \\LAB-DC01\IPC$ /user:BLUELAB\adm-mxnx <contraseña-incorrecta>
```

Resultado en el cliente: `Error de sistema 1326. El nombre de usuario o la contraseña no son correctos.`

## 3. Evidencia

**Acceso exitoso (4624) extraído del XML del evento:**

| Campo | Valor |
|---|---|
| Usuario | `adm-mxnx` |
| LogonType | `3` (red) |
| IP | `172.31.250.20` |
| Auth | `Kerberos` |

**Fallos: primera búsqueda** → ni `4625` ni `4771` asociados a las pruebas.

**Diagnóstico de la causa:**

```powershell
auditpol /get /category:*
```

Las subcategorías de Kerberos auditaban **solo Aciertos**. Además, la GPO de auditoría existente estaba vinculada a la OU de estaciones de trabajo, no a Domain Controllers.

**Corrección:** GPO nueva `GPO-DC-AUDITORIA-BLUE-TEAM`, vinculada a `Domain Controllers`, con Aciertos y Errores en:
- Validación de credenciales
- Servicio de autenticación Kerberos
- Operaciones de vales de servicio Kerberos

Verificado nuevamente con `auditpol`.

**Repetición de la prueba → Event ID 4771 en `LAB-DC01`:**

| Campo | Valor | Interpretación |
|---|---|---|
| TargetUserName | `adm-mxnx` | Cuenta objetivo |
| ServiceName | `krbtgt/BLUELAB` | Solicitud de TGT |
| Status | `0x18` | Preautenticación fallida: contraseña incorrecta |
| PreAuthType | `2` | |
| IpAddress | `::ffff:172.31.250.20` | Origen: `LAB-CLIENT01` |

## 4. Análisis

- **Por qué no había evidencia:** la fuente no estaba configurada para generarla, no porque no hubiera ocurrido nada.
- **Regla operativa:** si no aparece la evidencia esperada, primero se verifica que la fuente esté configurada para producirla y después se descarta el evento.
- **4771 versus 4625:** en un dominio, el fallo de contraseña en Kerberos se registra en el DC como 4771; buscar solo 4625 deja fuera estos intentos.

## 5. MITRE ATT&CK

- **T1110.001 — Password Guessing:** técnica simulada (intento con contraseña incorrecta).
- **T1110.003 — Password Spraying:** no realizada; es el escenario que esta brecha habría dejado sin detectar.

## 6. Detección

```text
Fuente: Security Log del DC (Event ID 4771, Status 0x18)
SI una misma IP genera 4771 contra ≥ 5 cuentas distintas en 10 min
ENTONCES alerta: posible password spraying
SI una misma cuenta acumula ≥ 10 eventos 4771 en 10 min
ENTONCES alerta: posible fuerza bruta contra la cuenta
```

Umbrales iniciales de laboratorio, a ajustar según la política de bloqueo de cuentas. **Pendiente:** validarlo en Wazuh con el DC como agente.

## 7. Respuesta

- **En el laboratorio:** se corrigió la causa raíz (auditoría del DC).
- **En un entorno real**, ante 4771 masivos: identificar la IP de origen, revisar si alguna cuenta tuvo un 4624 posterior desde esa misma IP (señal de éxito tras los intentos), bloquear el origen y forzar el cambio de contraseña de las cuentas afectadas.

## 8. Lecciones

- La visibilidad se valida **antes** de necesitarla: provocar el evento y comprobar que llega.
- Una GPO vinculada a la OU equivocada no falla con un error; simplemente no se aplica.
- Extraer campos del XML del evento (`[xml]$_.ToXml()`) es más confiable que leer el mensaje de texto.

Guía completa: [lab 06](../02-laboratorio/06-autenticacion-4624-4771-gpo-dc.md) · [lab 08](../02-laboratorio/08-gpo-auditoria-domain-controllers.md)
