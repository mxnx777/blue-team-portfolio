# Autoevaluación — sesión 04-11-2026
**Instrucciones**: marca una alternativa por pregunta, sin ejecutar comandos primero. Al terminar, compara con `respuestas.md`. 1 punto por pregunta, 20 puntos totales. Objetivo: >=16. En las equivocadas, explica qué evidencia te haría cambiar de opinión.

## Linux básico
1. ¿Qué comando confirma el directorio actual?
   A) pwd  B) ls  C) whoami  D) mkdir
2. Si necesitas mostrar también archivos cuyo nombre inicia por punto:
   A) ls -l  B) ls -a  C) cd -a  D) cat -a
3. ¿Qué comando copia `note` a `Documents` sin borrar el original?
   A) cd note Documents/  B) mv note Documents/  C) cp note Documents/  D) rm note
4. En el prompt `tryhackme@linux3:~$`, el símbolo `~` indica:
   A) raíz /  B) home del usuario  C) carpeta /tmp  D) usuario root
5. `cat flag.txt` falla cuando existe `.flag.txt`. La causa:
   A) son nombres distintos  B) falta sudo  C) faltó SSH  D) falla HTTP

## Identidad y permisos
6. En `-rw-r-----`, los permisos para «otros» son:
   A) lectura  B) lectura/escritura  C) ejecución  D) ninguno
7. ¿Qué comando cambia mediante credenciales legítimas a la sesión de `user2`?
   A) login user2  B) su -l user2  C) cd user2  D) fg user2
8. `tryhackme is not in the sudoers file` significa:
   A) no hay conexión  B) archivo oculto  C) el usuario no está autorizado para sudo  D) no existe bash

## Transferencia y procesos
9. `wget http://IP:8000/.flag.txt` se ejecuta para:
   A) crear servidor web  B) descargar recurso HTTP  C) cambiar usuario  D) habilitar cron
10. Si el servidor web está en AttackBox y haces wget a la IP de Lab Machine, el problema probable es:
    A) puerto en la máquina equivocada  B) cambiar extensión  C) contraseña sudo  D) grupo Domain Admins
11. ¿Qué comando lista muchos procesos con usuario, PID y línea de comando?
    A) touch  B) ps aux  C) pwd  D) scp
12. ¿Cuál diferencia es correcta?
    A) Ctrl+Z termina definitivamente  B) fg elimina trabajo  C) Ctrl+Z suspende y fg reanuda en primer plano  D) jobs lista todos los procesos del sistema
13. ¿Qué señal solicita normalmente una terminación ordenada?
    A) SIGKILL  B) SIGTERM  C) SIGSTOP  D) SIGCONT

## Automatización, paquetes y logs
14. `@reboot /var/opt/processes.sh` indica:
    A) cada minuto  B) al arrancar cron/sistema  C) cada 12 h  D) cada domingo
15. ¿Qué hace `apt update`?
    A) borra paquetes  B) actualiza índices/listas disponibles  C) actualiza siempre todos los paquetes  D) muestra usuarios
16. En Apache, un registro `"GET /catsanddogs.jpg HTTP/1.1" 200` significa:
    A) borrado exitoso  B) solicitud GET con respuesta 200  C) conexión SSH  D) cambio de GPO
17. Si `access.log` está restringido pero `access.log.1` tiene permisos `777`, ¿cuál afirmación es prudente?
    A) todos son maliciosos  B) se demuestra borrado  C) el rotado permite accesos excesivos y hay riesgo para la integridad  D) solo root puede leerlo

## Active Directory
18. ¿Qué objeto suele tener nombre `TOM-PC$`?
    A) cuenta de máquina  B) OU  C) impresora  D) GPO
19. Si necesitas agrupar usuarios de Quality Assurance para aplicar GPO por ubicación organizativa, usas:
    A) Backup Operators  B) OU  C) archivo de log  D) PID
20. ¿Qué grupo tiene privilegios administrativos amplios en el dominio por defecto?
    A) Domain Users  B) Domain Computers  C) Domain Admins  D) Guests

## Casos de razonamiento (sin puntaje)
- Un proceso tiene `STAT=T`. ¿Qué puede estar ocurriendo? ¿Cómo comprobarías antes de actuar?
- Un servidor presenta un servicio HTTP inesperado: ¿qué revisarías primero para determinar máquina, proceso, puerto y archivos expuestos?
- Un usuario solicita acceso a carpetas: ¿en qué difiere ubicarlo en una OU de incorporarlo a un grupo con permisos NTFS?
- Un log devuelve Permission denied: ¿qué evidencias consultarías antes de escalar una solicitud de acceso?
