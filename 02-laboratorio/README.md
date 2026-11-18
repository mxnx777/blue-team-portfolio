# Laboratorio BLUELAB

Construcción y ejercicios del laboratorio propio: Active Directory, auditoría de eventos Windows, Sysmon, Wazuh y actividad controlada desde Kali. Orden cronológico.

| Archivo | Tema |
|---|---|
| [01-bluelab-active-directory.md](01-bluelab-active-directory.md) | Laboratorio Blue Team 01 — Active Directory, DNS y cliente Windows |
| [02-ou-usuarios-grupos-gpo-minimo-privilegio.md](02-ou-usuarios-grupos-gpo-minimo-privilegio.md) | Guía práctica — OU, usuarios, grupos, GPO y mínimo privilegio |
| [03-auditoria-eventos-windows.md](03-auditoria-eventos-windows.md) | Guía práctica — Auditoría avanzada y análisis de eventos Windows |
| [04-sysmon-procesos-red-correlacion.md](04-sysmon-procesos-red-correlacion.md) | Guía práctica — Sysmon: procesos, red y correlación de eventos |
| [05-validacion-kerberos-klist-cifs-sysmon.md](05-validacion-kerberos-klist-cifs-sysmon.md) | Validación práctica de Kerberos con `klist`, CIFS/SMB, Sysmon y eventos 4768/4769 |
| [06-autenticacion-4624-4771-gpo-dc.md](06-autenticacion-4624-4771-gpo-dc.md) | Autenticación en Active Directory: 4624 LogonType 3, fallo Kerberos 4771 y auditoría por GPO |
| [07-logons-windows-4624-4625-correlacion.md](07-logons-windows-4624-4625-correlacion.md) | Guía práctica — Logons Windows 4624/4625 y correlación con Kerberos |
| [08-gpo-auditoria-domain-controllers.md](08-gpo-auditoria-domain-controllers.md) | Guía práctica — Crear una GPO de auditoría para Domain Controllers (GUI + comandos) |
| [09-cierre-etapa-windows-ad-sysmon.md](09-cierre-etapa-windows-ad-sysmon.md) | Cierre de etapa — Windows, Active Directory, Sysmon, Kerberos y plan de repetición |
| [10-wazuh-siem-agente-windows-sysmon.md](10-wazuh-siem-agente-windows-sysmon.md) | Wazuh SIEM: instalación, agente Windows, Sysmon y Threat Hunting |
| [11-kali-nmap-smb-firewall-drop.md](11-kali-nmap-smb-firewall-drop.md) | Kali + Nmap + SMB + Firewall DROP |
| [12-reconocimiento-ad-ldap-cuenta-estandar.md](12-reconocimiento-ad-ldap-cuenta-estandar.md) | Reconocimiento de Active Directory con cuenta estándar (LDAP) |
| [13-cuentas-smb-ldap-kerberos-configuracion-segura.md](13-cuentas-smb-ldap-kerberos-configuracion-segura.md) | BLUELAB — Cuentas, SMB, LDAP y Kerberos |
| [14-guia-operativa-blueteam-ad.md](14-guia-operativa-blueteam-ad.md) | Guía operativa Blue Team / Active Directory |
| [15-puertos-pid-servicio-firewall-nmap-smb.md](15-puertos-pid-servicio-firewall-nmap-smb.md) | Caso práctico SOC: puerto → PID → proceso → servicio → firewall → evidencia de DROP |
