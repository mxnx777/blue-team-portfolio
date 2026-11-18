Propósito:
Checklist operativo, no teórico.

Simulación de Incidente — Checklist
1️⃣ Identificación

¿Qué servidor?

¿Qué servicio?

¿Qué cambió respecto al baseline?

2️⃣ Contención

¿L1, L2 o L3?

¿A quién se notifica?

¿Se documenta hora y acción?

3️⃣ Análisis

Revisar:

SMB

WinRM

IIS

Usuarios

Comparar con inventario previo

4️⃣ Recuperación

¿Hay backup válido?

¿Restore probado?

¿Tiempo de recuperación?

5️⃣ Post-mortem

¿Qué falló?

¿Qué se mejora?

¿Qué se documenta?

# Checklist Post Corte Eléctrico – SQL Server

- [ ] Servidor encendido correctamente
- [ ] Servicio SQL Server (Engine) en ejecución
- [ ] Servicio SQL Server Agent en ejecución
- [ ] Verificar último backup ejecutado
- [ ] Revisar logs de errores (Event Viewer / SQL Agent)

## Observación
Si SQL Server Agent no está en ejecución, los backups y tareas automáticas
NO se ejecutarán.
