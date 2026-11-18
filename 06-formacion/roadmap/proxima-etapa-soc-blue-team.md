# 11 — Próxima etapa SOC / Blue Team (PENDIENTE DE EJECUCIÓN)

Estado: **NO ejecutado todavía**  
Laboratorio: `BLUELAB / bluelab.test`

Este documento separa claramente lo ya realizado de lo que viene a continuación.

La etapa anterior dejó funcionando y validado:

- Wazuh SIEM
- agente Windows
- Sysmon
- Threat Hunting
- Wazuh Archives
- correlación de procesos
- Event ID 1 / Event ID 5
- Windows Security 4688
- ProcessGuid
- triage de PowerShell
- MITRE ATT&CK
- clasificación de eventos y alertas

---

## 1. Objetivo de la próxima etapa

Pasar desde investigación manual de procesos a casos SOC más cercanos a un entorno real.

Flujo objetivo:

```text
generar comportamiento controlado
→ recolectar telemetría
→ detectar
→ investigar
→ correlacionar
→ mapear MITRE
→ clasificar
→ documentar evidencia
→ crear detección/regla cuando corresponda
```

---

## 2. Sysmon Event ID 3 — conexiones de red

### Qué se quiere aprender

Investigar conexiones de red iniciadas por procesos.

### Qué revisar

- proceso que inicia la conexión
- usuario
- IP origen
- IP destino
- puerto origen
- puerto destino
- protocolo
- hostname/dominio
- ProcessGuid
- relación con el proceso padre

### Meta SOC

Poder responder:

```text
qué proceso salió a la red
→ hacia dónde
→ por qué puerto
→ bajo qué usuario
→ y si era esperado
```

---

## 3. DNS y resolución de nombres

### Objetivo

Aprender a investigar consultas DNS desde el endpoint.

### Casos

- resolución normal
- dominios desconocidos
- múltiples consultas
- proceso que origina la consulta
- correlación DNS + conexión de red

### Pregunta de análisis

```text
¿qué proceso consultó qué dominio y qué ocurrió después?
```

---

## 4. Creación de usuarios

### Objetivo

Detectar y analizar creación de cuentas.

### Revisar

- usuario que ejecuta la acción
- cuenta creada
- host
- hora
- proceso
- CommandLine
- privilegios
- eventos Windows relacionados
- MITRE correspondiente

### Resultado esperado

Distinguir entre:

- alta administrativa legítima
- automatización
- comportamiento sospechoso

---

## 5. Cambios en grupos privilegiados

### Objetivo

Investigar modificaciones en grupos como administradores locales o grupos privilegiados del dominio.

### Revisar

- quién agregó al usuario
- qué usuario fue agregado
- grupo afectado
- equipo/DC
- hora
- evento Windows
- proceso asociado

### Meta

Entender por qué un cambio de privilegio puede ser más relevante que la simple creación de una cuenta.

---

## 6. Fallos de autenticación

### Objetivo

Aprender a leer intentos fallidos y patrones de autenticación.

### Casos

- contraseña incorrecta
- usuario inexistente
- múltiples fallos
- origen del intento
- correlación con logon exitoso posterior

### Telemetría principal

- Security Log
- eventos de autenticación del cliente
- eventos del Domain Controller

### Meta SOC

Distinguir:

- error humano
- credencial antigua
- servicio mal configurado
- comportamiento sospechoso

---

## 7. Persistencia controlada

### Objetivo

Generar ejemplos autorizados dentro del laboratorio para estudiar persistencia.

### Enfoque

No se busca "atacar por atacar".

Se busca:

```text
crear comportamiento controlado
→ observar eventos
→ entender qué detecta Wazuh
→ revisar MITRE
→ hacer triage
```

### Meta

Reconocer indicadores de persistencia y diferenciarlos de configuraciones administrativas legítimas.

---

## 8. Comparar LAB-CLIENT01 con LAB-DC01

### Objetivo

Agregar el Domain Controller como segunda fuente importante de telemetría.

Comparar:

```text
LAB-CLIENT01
vs
LAB-DC01
```

### Qué cambia

En el DC se espera mayor valor en:

- autenticación
- Kerberos
- cuentas
- grupos
- políticas
- eventos de Active Directory
- cambios administrativos

### Meta

Empezar a correlacionar endpoint + controlador de dominio.

---

## 9. Correlación cliente + DC

Ejemplo conceptual:

```text
acción desde LAB-CLIENT01
→ evento local
→ autenticación en LAB-DC01
→ evento del DC
→ alerta/detección en Wazuh
→ investigación conjunta
```

### Objetivo

Dejar de mirar un solo host y empezar a reconstruir actividad entre sistemas.

---

## 10. Reglas Wazuh personalizadas

### Objetivo

Dejar de depender únicamente de reglas por defecto.

Crear reglas propias basadas en comportamiento observado en el laboratorio.

### Metodología

1. generar evento;
2. confirmar que llega;
3. entender campos;
4. definir condición;
5. crear regla;
6. probar;
7. validar falsos positivos;
8. documentar.

### Meta

Poder demostrar en portafolio:

```text
"identifiqué un caso de uso y construí una detección"
```

---

## 11. MITRE ATT&CK aplicado

En cada caso documentar:

- tactic
- technique
- sub-technique si corresponde
- fuente de datos
- evento
- regla
- contexto
- conclusión

### Meta

No memorizar MITRE.

Usarlo como lenguaje para describir comportamiento.

---

## 12. Playbooks SOC N1

Crear playbooks simples para investigaciones frecuentes.

Ejemplos:

- PowerShell sospechoso
- creación de usuario
- cambio de grupo privilegiado
- múltiples logons fallidos
- proceso con conexión de red
- DNS sospechoso
- ejecución desde ubicación inusual

Cada playbook debe incluir:

- qué revisar primero
- campos mínimos
- evidencia
- cuándo cerrar
- cuándo escalar

---

## 13. Formato estándar de cada ejercicio futuro

Cada caso nuevo debe documentarse así:

### Objetivo

Qué comportamiento se quiere observar.

### Acción controlada

Qué se generó en el laboratorio.

### Fuente

- Sysmon
- Security
- DC
- Wazuh

### Evento

Event ID / regla / canal.

### Evidencia

Campos principales.

### Correlación

Eventos relacionados.

### MITRE

Tactic + Technique.

### Triage

Qué significa.

### Clasificación

- benigno esperado
- verdadero positivo no malicioso
- sospechoso
- incidente

### Conclusión

Qué aprendimos y cómo lo explicaría un analista SOC.

---

## 14. Orden recomendado de ejecución

1. Sysmon Event ID 3 — Network Connection
2. DNS
3. fallos de autenticación
4. creación de usuario
5. cambios de grupos
6. incorporar LAB-DC01 a Wazuh
7. correlación cliente + DC
8. persistencia controlada
9. reglas Wazuh personalizadas
10. playbooks SOC N1

---

## 15. Resultado esperado de esta etapa

Al terminar, el laboratorio debería demostrar capacidad para:

- observar endpoint y DC;
- investigar procesos;
- investigar red;
- investigar DNS;
- investigar autenticación;
- analizar cuentas y privilegios;
- correlacionar múltiples hosts;
- mapear MITRE;
- crear reglas propias;
- realizar triage repetible;
- documentar casos en formato profesional.

---

## Estado actual

```text
PENDIENTE
```

Este archivo es roadmap.

No debe marcarse como completado hasta ejecutar y documentar cada bloque con evidencia real.
