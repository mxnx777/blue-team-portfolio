# 10 — Playbook SOC: Threat Hunting y triage con Wazuh + Sysmon

Fecha de consolidación: 30-10-2026  
Laboratorio: BLUELAB / bluelab.test

## Objetivo

Dejar una guía repetible de la metodología usada en esta etapa del laboratorio para investigar procesos Windows desde Wazuh, correlacionar Sysmon y Windows Security, usar Wazuh Archives y clasificar correctamente una detección.

La meta es responder siempre:

- qué ocurrió;
- quién lo ejecutó;
- desde qué proceso padre;
- con qué argumentos;
- qué técnica MITRE representa;
- si la detección es válida;
- si realmente existe un incidente;
- qué evidencia respalda la conclusión.

---

## 1. Regla central

EVENTO != ALERTA != INCIDENTE

Un evento es telemetría.

Una alerta es una coincidencia con una regla.

Un incidente exige contexto y evidencia suficiente para justificar respuesta.

Por eso una detección puede ser técnicamente correcta y aun así corresponder a administración legítima, una herramienta de seguridad o una prueba autorizada.

---

## 2. Fuentes de telemetría

### Sysmon

Canal:

Microsoft-Windows-Sysmon/Operational

Eventos usados:

- Event ID 1: creación de proceso.
- Event ID 5: término de proceso.

### Windows Security

Evento importante:

- Event ID 4688: creación de nuevo proceso mediante auditoría nativa.

### Diferencia práctica

Windows Security entrega auditoría nativa.

Sysmon entrega telemetría de proceso enriquecida, normalmente incluyendo:

- Image
- CommandLine
- ParentImage
- ParentCommandLine
- User
- IntegrityLevel
- Hashes
- ProcessGuid
- ParentProcessGuid

---

## 3. Filtro base para investigar procesos Sysmon

En Threat Hunting o Discover:

- data.win.system.channel = Microsoft-Windows-Sysmon/Operational
- data.win.system.eventID = 1

### ¿Qué buscamos?

Quedarnos con eventos de creación de procesos registrados por Sysmon.

### ¿Por qué?

Porque el triage parte reconstruyendo qué proceso se ejecutó y cómo nació.

---

## 4. Si se borra un filtro

Eliminar un filtro NO elimina:

- logs;
- alertas;
- índices;
- eventos;
- archivos.

Solo cambia lo que Wazuh/OpenSearch muestra.

Para reconstruir la búsqueda:

1. Restaurar un rango temporal útil.
2. Agregar nuevamente el canal Sysmon.
3. Agregar eventID = 1.
4. Aplicar filtros adicionales según el proceso investigado.

---

## 5. Campos mínimos de triage

Revisar siempre:

- data.win.eventdata.image
- data.win.eventdata.commandLine
- data.win.eventdata.parentImage
- data.win.eventdata.parentCommandLine
- data.win.eventdata.user
- data.win.eventdata.integrityLevel
- data.win.eventdata.processId
- data.win.eventdata.parentProcessId
- data.win.eventdata.processGuid
- data.win.eventdata.parentProcessGuid
- data.win.eventdata.hashes
- data.win.system.eventID
- data.win.system.channel
- rule.id
- rule.level
- rule.description
- rule.mitre.id
- rule.mitre.tactic
- rule.mitre.technique

Pregunta mental:

QUIÉN → ejecutó QUÉ → desde DÓNDE → con QUÉ argumentos → mediante QUÉ proceso padre → CUÁNDO → en QUÉ host → y POR QUÉ disparó una regla.

---

## 6. Caso: Discovery con net user

Se observó la detección:

- rule.description: Discovery activity executed
- rule.id: 92031
- rule.level: 3
- MITRE: T1087 Account Discovery

Cadena observada:

net.exe → net1.exe → net1 user

Contexto relevante:

- User: NT AUTHORITY\SYSTEM
- CurrentDirectory relacionado con el agente Wazuh.

### Interpretación

La detección es válida porque el comportamiento corresponde a enumeración de cuentas.

Sin embargo, el contexto indicó actividad coherente con una evaluación del agente, por lo que se clasificó provisionalmente como benigno esperado.

### Lección

Una detección válida no implica un incidente.

---

## 7. Caso: PowerShell + eliminación de archivo temporal

Se observó una alerta asociada a File Deletion, MITRE T1070.004.

La cadena de procesos mostró:

wazuh-agent.exe → powershell.exe → secedit / lectura de política → eliminación de archivo temporal

### Interpretación

La regla detectó correctamente una eliminación realizada desde PowerShell.

El contexto mostró:

- proceso padre Wazuh;
- ejecución como SYSTEM;
- consulta de configuración de seguridad;
- eliminación posterior del archivo temporal generado.

Clasificación:

Benigno esperado / actividad del agente.

---

## 8. Correlación temporal

Se utilizó View surrounding documents.

### ¿Para qué?

Para observar eventos anteriores y posteriores y reconstruir la secuencia completa.

Esto permitió distinguir entre:

- una acción aislada que parecía sospechosa;
- y una secuencia coherente iniciada por una herramienta de seguridad.

Lección:

evento aislado != historia completa.

---

## 9. Windows Security 4688 vs Sysmon Event ID 1

Se comprobó que un mismo proceso puede aparecer en ambas fuentes.

Windows Security 4688:
- auditoría nativa;
- NewProcessName;
- CommandLine;
- ParentProcessName;
- SubjectUserName.

Sysmon Event ID 1:
- Image;
- CommandLine;
- User;
- IntegrityLevel;
- Hashes;
- ProcessGuid;
- ParentProcessGuid;
- metadatos del binario.

### Lección

Múltiples eventos no significan múltiples ataques.

Pueden ser distintas fuentes describiendo el mismo proceso.

---

## 10. Wazuh Archives

Se habilitó la conservación e indexación de telemetría que no necesariamente genera alertas.

### ¿Por qué?

Porque buscar solamente en wazuh-alerts-* deja fuera eventos válidos que no dispararon reglas.

Con Archives se validó el flujo:

Windows → Sysmon → Wazuh Agent → Manager → archives.json → Filebeat → Indexer → wazuh-archives-* → Discover

### Resultado

Se comprobó que procesos como ipconfig.exe existían en la telemetría aunque no aparecieran como una alerta destacada.

---

## 11. Correlación por ProcessGuid

Se siguió una misma instancia de proceso entre:

- Event ID 1: creación.
- Event ID 5: terminación.

La correlación se hizo usando el mismo ProcessGuid.

### ¿Por qué es mejor que usar solo PID?

El PID puede reutilizarse.

ProcessGuid permite identificar con mayor precisión una instancia concreta de proceso.

---

## 12. Caso Discovery controlado

Se estudiaron procesos derivados de comandos de descubrimiento autorizados dentro del laboratorio.

Árbol observado:

powershell.exe
  ├─ net.exe
  │   └─ net1.exe
  └─ whoami.exe

Mapeo MITRE utilizado:

- Account Discovery.
- Permission Groups Discovery.
- System Owner/User Discovery según contexto.

### Lección

Nunca asumir relaciones padre-hijo.

Verificar siempre:

- ParentImage;
- ParentProcessGuid;
- ParentProcessId.

---

## 13. Caso PowerShell EncodedCommand

Se realizó una prueba autorizada en el laboratorio con PowerShell codificado.

Wazuh generó una detección de nivel alto asociada a:

- PowerShell;
- T1059.001;
- uso de EncodedCommand.

El contenido fue decodificado sin ejecutar nada desconocido y resultó ser una prueba benigna.

### Clasificación

- Verdadero positivo de detección.
- No incidente malicioso.

### Lección

Una regla puede detectar correctamente una técnica sospechosa aunque el contenido concreto sea benigno y autorizado.

Los comandos exactos y la evidencia están conservados en el documento 09 del laboratorio.

---

## 14. Método de triage repetible

Para cada alerta:

1. Confirmar host.
2. Confirmar usuario.
3. Confirmar fecha y hora.
4. Identificar proceso.
5. Leer CommandLine.
6. Identificar proceso padre.
7. Leer ParentCommandLine.
8. Revisar IntegrityLevel.
9. Revisar PID y GUID.
10. Revisar hash si corresponde.
11. Revisar regla Wazuh.
12. Revisar MITRE.
13. Buscar eventos cercanos.
14. Buscar el mismo ProcessGuid.
15. Comparar con Security 4688 si existe.
16. Determinar si corresponde a:
   - actividad esperada;
   - administración;
   - herramienta de seguridad;
   - prueba controlada;
   - actividad sospechosa;
   - actividad maliciosa.
17. Documentar evidencia antes de cerrar.

---

## 15. Plantilla de informe SOC

HOST:
USUARIO:
FECHA/HORA:

ALERTA:
RULE ID:
RULE LEVEL:

MITRE:
TACTIC:
TECHNIQUE:

IMAGE:
COMMAND LINE:

PARENT IMAGE:
PARENT COMMAND LINE:

PROCESS ID:
PROCESS GUID:

EVIDENCIA RELACIONADA:

ANÁLISIS:

CLASIFICACIÓN:
- benigno esperado
- falso positivo
- verdadero positivo no malicioso
- sospechoso
- incidente

ACCIÓN:

---

## 16. Errores y aprendizajes operativos

### Filtro borrado

No se pierde telemetría. Solo se reconstruye la búsqueda.

### Rango temporal incorrecto

Puede mostrar “No results match your search criteria” aunque los eventos existan.

### Buscar solo wazuh-alerts-*

Puede ocultar telemetría válida. Para hunting completo se usa wazuh-archives-*.

### Confundir 4688 con Sysmon 1

Ambos representan creación de procesos, pero desde fuentes distintas.

Siempre revisar:

- channel;
- providerName;
- eventID.

---

## 17. Capacidad demostrada

En esta etapa quedó demostrado que el laboratorio permite:

- recolectar Sysmon;
- recibirlo en Wazuh;
- investigar Event ID 1;
- correlacionar Event ID 1 y 5;
- distinguir Security 4688 de Sysmon 1;
- trabajar con Wazuh Alerts;
- trabajar con Wazuh Archives;
- reconstruir árboles de procesos;
- usar ProcessGuid;
- interpretar MITRE ATT&CK;
- investigar PowerShell codificado de forma segura;
- separar evento, alerta e incidente;
- documentar conclusiones con evidencia.

---

## 18. Próximo nivel

La metodología a repetir será:

comportamiento controlado → detección → investigación → MITRE → evidencia → clasificación → documentación

Siguientes casos recomendados:

- Sysmon Event ID 3 y conexiones de red;
- DNS;
- creación de usuarios;
- cambios de grupos privilegiados;
- fallos de autenticación;
- persistencia controlada;
- comparación LAB-CLIENT01 vs LAB-DC01;
- reglas Wazuh personalizadas;
- playbooks SOC N1.

---

## Regla mental final

No mirar solo la alerta. Reconstruir la historia.

Eso es lo que transforma “ver Wazuh” en hacer triage de verdad.
