# BLUELAB — Cuentas, SMB, LDAP y Kerberos

## Objetivo
Demostrar, en un laboratorio Windows/Active Directory controlado, cómo una cuenta de bajo privilegio puede entregar contexto útil a un atacante y cómo un analista Blue Team puede investigar esa actividad mediante Windows Event Logs, Sysmon, firewall y Wazuh.

## Topología
- `LAB-DC01` — `172.31.250.10` — Domain Controller (`bluelab.test`)
- `LAB-CLIENT01` — `172.31.250.20` — Windows + Sysmon + Wazuh Agent
- `WAZUH-SRV01` — `172.31.250.30` — Wazuh SIEM
- `KALI-ATTACK01` — `172.31.250.40` — host de pruebas ofensivas
- Segmento: `LAB_ISOLADO` — `172.31.250.0/24`

## Siglas esenciales
- **SIEM** — Security Information and Event Management
- **SMB** — Server Message Block
- **LDAP** — Lightweight Directory Access Protocol
- **AD** — Active Directory
- **DC** — Domain Controller
- **SPN** — Service Principal Name
- **TGS** — Ticket Granting Service ticket
- **SOC** — Security Operations Center
- **EDR** — Endpoint Detection and Response
- **XDR** — Extended Detection and Response

## Flujo
1. Configurar Kali dentro del segmento aislado.
2. Verificar conectividad y rutas.
3. Reconocer puertos y servicios.
4. Comparar visión remota con listeners locales.
5. Relacionar `puerto → PID → proceso → servicio`.
6. Validar el efecto del firewall con logs.
7. Analizar SMB y autenticaciones.
8. Usar una cuenta estándar para consultas LDAP.
9. Enumerar usuarios, grupos, equipos y relaciones.
10. Revisar SPN/Kerberos como riesgo de cuentas de servicio.
11. Correlacionar evidencia con Windows/Wazuh.
12. Mitigar y volver a comprobar.

## Hallazgos

### 1. `filtered` no significa `closed`
Kali observó TCP/445 como `filtered` mientras Windows mostraba el servicio en `LISTENING`. El firewall confirmó tráfico descartado desde `172.31.250.40` hacia `172.31.250.20:445`.

**Conclusión:** el servicio existía, pero un control del host impedía el acceso remoto.

### 2. Puerto → proceso → servicio
Modelo:

```text
puerto → PID → proceso → servicio
```

Ejemplo observado:

```text
5040 → PID 4516 → svchost.exe → CDPSvc
```

### 3. Postura SMB
- Enumeración anónima rechazada.
- SMB Signing requerido.
- SMBv1 no observado.
- SMB2/SMB3 disponibles.

**Conclusión:** un puerto abierto no equivale a vulnerabilidad.

### 4. Cuenta estándar comprometida
Una cuenta estándar puede autenticarse contra servicios permitidos, enumerar recursos accesibles y consultar bastante información de Active Directory mediante LDAP. No necesita ser Domain Admin para aportar información útil para construir una ruta de ataque.

## LDAP — modelo mental
```text
Active Directory = directorio
LDAP             = protocolo para consultarlo
ldapsearch       = herramienta para formular consultas
```

## Configuración de cuentas — lección principal
- Aplicar mínimo privilegio.
- Separar cuenta estándar y administrativa.
- Controlar grupos administrativos.
- Tratar cuentas de servicio como identidades sensibles.
- Usar contraseñas fuertes y alcance limitado.
- Revisar SPN, memberships y recursos accesibles.
- Evitar privilegios asignados por comodidad.

El riesgo de una credencial comprometida depende de **todo lo que esa identidad puede alcanzar**, no solo de su nombre.

## Eventos Windows importantes
- `4625` — Failed Logon
- `4624` — Successful Logon
- `Logon Type 3` — Network logon
- `4769` — Kerberos service ticket requested

## Errores útiles
- Prefijo `/32` en vez de `/24`.
- `-pn` en vez de `-Pn` en Nmap.
- Buscar reglas por nombres localizados.
- Sintaxis SMB mezclando dominio y usuario.
- `ldapsearch` multilínea mal interpretado.
- Intentar resolver varias capas simultáneamente.

**Lección:** una variable por vez y evidencia antes de concluir.

## Método SOC repetible
```text
pregunta
→ hipótesis
→ comprobación local
→ comprobación remota
→ evidencia
→ correlación
→ conclusión
→ mitigación
→ reprueba
```

## Valor para portfolio
Este bloque demuestra práctica con TCP/IP, servicios Windows, firewall, SMB, Active Directory, LDAP, autenticación, Kerberos, triage, Wazuh/SIEM y documentación técnica.

## Próximo paso
Continuar con BLUELAB + HTB/TryHackMe y convertir cada caso relevante en una guía de estudio limpia y un caso Markdown corto y defendible en GitHub.