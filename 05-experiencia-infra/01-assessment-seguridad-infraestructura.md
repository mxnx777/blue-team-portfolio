# Assessment de seguridad de infraestructura — organización pequeña

**Tipo:** levantamiento real en entorno laboral (versión anonimizada)  
**Alcance:** red local, 3 servidores Windows Server, equipos de red y estaciones  
**Marco de referencia:** NIST CSF (Identificar → Proteger → Detectar → Responder → Recuperar)

## 1. Problema de partida

No existía inventario confiable, documentación de servidores ni evaluación de riesgos. El objetivo fue pasar de "creemos que tenemos esto" a **activos confirmados con evidencia**, y de ahí a riesgos priorizados.

## 2. Inventario verificado

- Escaneo de red para descubrir activos (IP, MAC, fabricante).
- **Verificación manual de cada hallazgo antes de incluirlo.** Ejemplo real: el escáner reportó una IP con MAC `00:00:00:00:00:00`. Verificación:

```cmd
arp -a | find "192.168.X.X"
ping 192.168.X.X
```

Sin entrada ARP ni respuesta → registro fantasma del escáner. Se documentó, pero **no se contó como activo**.

- Regla del inventario: solo activos confirmados; lo dudoso va a bitácora con el comando usado.

## 3. Evaluación por servidor

Para cada servidor se levantó, con comandos nativos de Windows:

| Área | Qué se verificó |
|---|---|
| Identidad | Dominio o workgroup, rol (`wmic computersystem get domain,domainrole,name`) |
| Sistema operativo | Versión, build, estado de soporte |
| Red | Puertos en escucha y **proceso dueño (PID)** de cada puerto |
| Servicios | Roles reales: SMB, IIS, SQL Server, WinRM |
| Cuentas | Usuarios locales, miembros de `Administrators` |
| Recursos compartidos | Shares SMB, permisos de share y NTFS |
| Protocolos | SMB1 habilitado o no |

## 4. Matriz de riesgos (resultado)

| Riesgo | Probabilidad | Impacto | Nivel |
|---|---|---|---|
| Servidor con sistema operativo fuera de soporte | Alta | Crítico | Alto |
| SMB1 activo en servidor heredado | Alta | Crítico | Alto |
| Sin gestión centralizada de identidades (sin AD) | Media | Alto | Alto |
| Backups sin prueba de restauración | Media | Alto | Medio-Alto |
| Exceso de cuentas administradoras locales | Media | Alto | Medio-Alto |
| WinRM sin restricción por IP de origen | Media | Medio | Medio |
| Firewall dependiente de reglas por defecto | Media | Medio | Medio |

## 5. Hardening priorizado

**Prioritario (alto impacto, bajo riesgo de romper producción):**
1. Retirar o reinstalar sistemas fuera de soporte. Esto reduce más riesgo que cualquier antivirus.
2. Deshabilitar SMB1.
3. Restringir WinRM por IP; mantenerlo solo si es necesario.
4. Reducir miembros de `Administrators`, eliminar cuentas genéricas, documentar cuentas de respaldo.

**Posterior (madurez):** centralización de logs, reglas explícitas de firewall, segmentación de red, Active Directory si el contexto lo permite.

## 6. Estado por fase de respuesta a incidentes

Registro honesto de lo ejecutado versus lo diseñado:

| Fase | Estado |
|---|---|
| Baseline | Ejecutado |
| Detección | Ejecutado (nivel 1) |
| Contención | Diseñada, no ejecutada |
| Erradicación | Parcial (vector identificado) |
| Recuperación | No ejecutada |
| Lecciones aprendidas | Se completa tras el hardening |

## 7. Qué aprendí

- Sin baseline no hay detección: primero saber qué es normal.
- Un hallazgo de escáner no es un activo hasta verificarlo.
- El riesgo más grande no siempre es técnico-sofisticado: un sistema sin soporte pesa más que cualquier configuración fina.
