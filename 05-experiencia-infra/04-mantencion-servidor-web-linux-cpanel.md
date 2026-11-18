# Mantención preventiva y revisión de seguridad de un servidor web Linux (cPanel/WHM)

**Tipo:** trabajo real en entorno laboral (versión anonimizada)  
**Fecha:** mayo 2026  
**Servidor:** VPS CentOS Linux 7.9 con cPanel/WHM 110, Apache, MySQL y ModSecurity (OWASP CRS)  
**Acceso:** root por SSH

## 1. Contexto

El sitio web institucional publica más de 12.000 documentos PDF por obligación legal de transparencia (~6,5 GB), accesibles mediante enlaces públicos. El disco estaba al 75 % y no había una política de respaldo verificada.

**Restricción principal:** cualquier cambio de ruta, nombre o permiso rompe enlaces públicos que la organización está obligada a mantener disponibles.

## 2. Levantamiento

```bash
cat /etc/centos-release            # versión del SO
df -h                              # uso de disco
free -h ; uptime                   # memoria y carga
du -sh /var/lib/mysql              # tamaño de las bases de datos
crontab -l                         # tareas programadas
grep -i backup /var/cpanel/backups/config   # política de respaldo real
/scripts/upcp --check              # estado de actualización de cPanel
```

## 3. Hallazgos

### H-01 (crítico): los respaldos parecían configurados, pero no se estaban generando

| Evidencia | Valor |
|---|---|
| `crontab -l` | `0 2 * * * /usr/local/cpanel/bin/backup`: el job diario existe |
| `/var/cpanel/backups/config` | `BACKUP_DAILY_ENABLE: 'yes'`, `BACKUP_DAILY_RETENTION: 5` |
| `/var/cpanel/backups/config` | **`BACKUPENABLE: 'no'`**: el interruptor general estaba apagado |
| `/backup` | 64 KB, sin respaldos útiles recientes |

**Interpretación:** quien revisara solo el cron o la retención diaria habría concluido que había respaldos. El interruptor general apagado anulaba todo. Es una falsa sensación de seguridad: ante un ransomware o una falla de disco, no había nada que restaurar.

**Recomendación:** política formal de respaldo con destino **externo** al servidor, retención definida y **prueba de restauración**. Antes de habilitarlo, evaluar el espacio, porque un respaldo local en el mismo disco al 75 % lo llenaría.

### H-02 (alto): sistema operativo al final de su vida útil

CentOS 7 terminó su soporte estándar. Queda registrado como **pendiente estratégico** planificar la migración a una distribución con soporte.

### H-03 (medio): espacio en disco ocupado por respaldos ZIP antiguos y corruptos

Había respaldos antiguos dentro de `public_html` y en la papelera. Un ZIP dentro de `public_html` es además un **riesgo de exposición**: se puede descargar si alguien adivina la URL (el mismo patrón que en [SOC-001](../01-casos-soc/SOC-001-reconocimiento-web-apache.md)).

## 4. Acciones ejecutadas

1. Inventario de los 12.264 PDF con ruta, tamaño y fecha (línea base para validar después de cualquier cambio).
2. Respaldo completo de la carpeta de transparencia, descargado fuera del servidor **antes** de borrar nada.
3. Validación de URLs públicas de muestra: `HTTP 200 OK`.
4. Eliminación de respaldos ZIP antiguos o corruptos de `.trash` y `public_html`.
5. Verificación de que no quedaran archivos mayores a 500 MB en la cuenta.
6. Revisión de actualización de cPanel (`upcp --check`: al día dentro de su rama; **sin** upgrade mayor ni cambio de rama, para no arriesgar producción).
7. Actualización del vendor de ModSecurity: OWASP CRS al día.

## 5. Resultado

| Métrica | Antes | Después |
|---|---|---|
| Disco usado / libre | 35 GB / 12 GB | 20 GB / 27 GB |
| Respaldo de transparencia | Ninguno verificado | Completo, fuera del servidor |
| Enlaces públicos | — | Muestra validada con HTTP 200 |

## 6. Qué aprendí

- **Verificar la configuración efectiva, no la aparente.** Un cron que corre no garantiza que el respaldo exista: hay que ir a mirar el archivo resultante.
- Antes de borrar, respaldar y medir; después del cambio, validar contra la línea base.
- En producción, "al día dentro de su rama" es una decisión consciente: un upgrade mayor sin ventana ni rollback es un riesgo mayor que el que resuelve.
