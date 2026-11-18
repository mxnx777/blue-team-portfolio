# BTLO Challenge: The Report (Red Canary 2022)

Resolución en español del desafío de Blue Team Labs Online (BTLO) basado en el reporte de amenazas de Red Canary 2022.

| # | Pregunta | Respuesta (Formato BTLO) | Explicación / Contexto |
|---|---|---|---|
| **Q1** | Nombre del ataque a la cadena de suministro relacionado con la librería de logs de Java a finales de 2021. | `Log4j` | Vulnerabilidad de ejecución remota de código en la librería de registros de Java expuesta en diciembre de 2021. |
| **Q2** | ID de la técnica MITRE ATT&CK que afectó a más del 50% de los clientes. | `T1059` | Corresponde a *Command and Scripting Interpreter*, afectando al 53.4% de los entornos. |
| **Q3** | Nombres de 2 vulnerabilidades pertenecientes a servidores Microsoft Exchange. | `ProxyLogon, ProxyShell` | Cadenas de vulnerabilidades explotadas ampliamente en 2021 para desplegar *web shells* y *ransomware*. |
| **Q4** | CVE de la vulnerabilidad zero-day de un controlador que permitió RCE y elevación a privilegios SYSTEM. | `CVE-2021-34527` | Asignado a *PrintNightmare*, falla grave en el servicio de cola de impresión (*print spooler*). |
| **Q5** | Menciona los 2 grupos de adversarios que abusan del posicionamiento SEO para obtener acceso inicial. | `Yellow Cockatoo, Gootkit` | Ambos grupos envenenan resultados de búsqueda para dirigir a las víctimas a descargas maliciosas. |
| **Q6** | Proceso padre que debe buscarse en reglas de detección para scripts `.js` maliciosos. | `wscript.exe` | *Windows Script Host* es el binario nativo que ejecuta scripts `.js` fuera del navegador. |
| **Q7** | Precursores utilizados por los afiliados del grupo de ransomware Conti para el acceso inicial. | `Qbot, Bazar, IcedID` | Familias de malware utilizadas por afiliados de acceso para entregar la carga útil de Conti. |
| **Q8** | Los 2 softwares desactualizados mencionados en el reporte que fueron objetivo principal de mineros de criptomonedas. | `JBoss, WebLogic` | Servidores de aplicaciones que, al no parchearse, fueron masivamente explotados por mineros como XMRig. |
| **Q9** | Nombre del grupo de ransomware que amenazó con realizar ataques DDoS si no se pagaba el rescate. | `Fancy Lazarus` | Grupo de extorsión que combinó amenazas de DDoS con solicitudes de pago en criptomonedas. |
| **Q10** | Medida de seguridad necesaria en conexiones RDP para protegerse contra ataques de ransomware. | `MFA` | La Autenticación de Múltiples Factores frena el acceso no autorizado vía RDP expuesto a internet. |

## Respuestas listas para copiar

1. **Q1:** `Log4j`
2. **Q2:** `T1059`
3. **Q3:** `ProxyLogon, ProxyShell`
4. **Q4:** `CVE-2021-34527`
5. **Q5:** `Yellow Cockatoo, Gootkit`
6. **Q6:** `wscript.exe`
7. **Q7:** `Qbot, Bazar, IcedID`
8. **Q8:** `JBoss, WebLogic`
9. **Q9:** `Fancy Lazarus`
10. **Q10:** `MFA`

## Aprendizajes clave

- MITRE ATT&CK sirve para conectar técnicas observadas con comportamiento adversario.
- Las vulnerabilidades conocidas pueden convertirse rápidamente en acceso inicial.
- Herramientas y binarios nativos de Windows pueden formar parte de cadenas de ataque.
- RDP sin MFA aumenta mucho el riesgo de acceso no autorizado.
- La lectura de reportes de amenazas ayuda a traducir incidentes reales a detecciones y controles defensivos.
