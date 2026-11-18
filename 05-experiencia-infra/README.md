# Experiencia en infraestructura (anonimizada)

Proyectos ejecutados en un entorno laboral real: una organización pequeña con servidores Windows Server, un servidor web Linux, red plana y sin Active Directory centralizado.

**Política de publicación:** se publica la **metodología y las decisiones técnicas**. Se omiten nombres de la organización, de proveedores, de usuarios, hostnames, IPs, MACs y cualquier dato que permita identificar el entorno. Los nombres que aparecen (`SRV-FILE01`, `SRV-APP01`, etc.) son genéricos.

| # | Proyecto | Qué demuestra |
|---|---|---|
| 01 | [Assessment de seguridad de infraestructura](01-assessment-seguridad-infraestructura.md) | Inventario verificado, evaluación por servidor, matriz de riesgos, priorización de hardening |
| 02 | [Migración de file server con mínimo privilegio](02-migracion-file-server-minimo-privilegio.md) | Plan de migración con rollback, modelo de grupos RW/RO, permisos SMB + NTFS |
| 03 | [Acceso temporal de proveedor y simulación de incidente](03-acceso-proveedor-y-simulacion-incidente.md) | Control y revocación de accesos, señales de movimiento lateral (4624+4672, WinRM), contención sin destruir evidencia |
| 04 | [Servidor web Linux (cPanel): mantención y revisión de seguridad](04-mantencion-servidor-web-linux-cpanel.md) | Linux en producción; hallazgo crítico de respaldos que aparentaban funcionar; liberación de espacio con línea base |
