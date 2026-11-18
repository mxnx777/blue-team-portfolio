# Ruta práctica de empleabilidad Blue Team / SOC — 2026

## Objetivo

Construir un perfil empleable para SOC / Blue Team combinando práctica propia, laboratorios guiados, investigación de incidentes y documentación técnica.

La meta no es completar cursos por completar. Cada actividad debe responder a una pregunta:

> ¿Esto aumenta la capacidad de investigar, detectar, explicar y responder a actividad sospechosa en un entorno real?

## Plataformas y función de cada una

### BLUELAB propio — eje principal
Usar el laboratorio Windows/AD propio para practicar:
- Active Directory (AD)
- LDAP
- SMB
- Kerberos
- Windows Event Logs
- Sysmon
- Wazuh SIEM
- Kali / Nmap
- firewall
- cuentas, grupos, permisos
- ataques controlados y detección

Objetivo: entender causa → efecto.

### HTB Academy — profundidad técnica
Usar para:
- Windows / AD
- Kerberos
- detección
- Splunk
- explotación
- técnicas ofensivas y defensivas avanzadas

Regla: no gastar Cubes sin revisar primero si el contenido aporta algo nuevo.

### TryHackMe — aprendizaje guiado y rápido
Usar para:
- networking
- SOC
- SIEM
- Incident Response
- Metasploit
- CVE actuales
- herramientas

### CyberDefenders / LetsDefend — casos SOC
Usar para entrenar:
- triage
- correlación
- análisis
- investigación
- decisión
- reporte

## Ciclo de estudio

1. Técnica nueva
2. Práctica ofensiva controlada
3. Observación defensiva
4. Caso SOC externo
5. Entrevista + GitHub

## Método operativo

Atacar → observar → correlacionar → explicar → detectar → mitigar → documentar.

## Modo analista

Antes de entregar un comando o respuesta:
1. formular hipótesis;
2. identificar evidencia necesaria;
3. seleccionar fuente/log;
4. ejecutar herramienta;
5. interpretar resultado;
6. decidir si es normal, sospechoso o incidente.

## Resultado esperado para entrevista

Ser capaz de explicar:
- qué ocurrió;
- cómo se detectó;
- qué evidencia lo demuestra;
- qué usuario/host/IP/proceso estuvo involucrado;
- qué técnica MITRE ATT&CK corresponde;
- qué mitigación aplicar;
- cuándo escalar.

## Siglas base a repetir

- SOC — Security Operations Center
- SIEM — Security Information and Event Management
- AD — Active Directory
- LDAP — Lightweight Directory Access Protocol
- SMB — Server Message Block
- EDR — Endpoint Detection and Response
- XDR — Extended Detection and Response
- IR — Incident Response
- DFIR — Digital Forensics and Incident Response
- IDS — Intrusion Detection System
- IPS — Intrusion Prevention System
- PCAP — Packet Capture
