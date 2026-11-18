# Bitácora de estudio — 04-11-2026
> Apuntes privados de aprendizaje. Entorno: salas autorizadas de TryHackMe. No mezclar con incidentes reales ni con infraestructura institucional. Sin datos personales, contraseñas ni IP temporales del laboratorio.

## Cómo repasar
1. Leer cada bloque y explicar verbalmente **qué buscamos → por qué → comando → evidencia → interpretación**.
2. Reproducir solamente en laboratorios autorizados.
3. Contestar `evaluacion.md` sin mirar `respuestas.md`.
4. Registrar errores y dudas para otra sesión.

## Avance observado
- Linux Fundamentals Part 2: **100% completada**.
- Linux Fundamentals Part 3: ejercicios de transferencia de archivos, procesos, Cron, paquetes y logs completados durante la sesión; no quedó registrado el 100 % global de la sala.
- Active Directory Basics: iniciada; revisión de AD DS, usuarios/equipos, grupos y OU; preguntas de Domain Admins y cuenta de equipo correctas; no se confirmó el cierre de toda la sala.
- Capability Score de plataforma observado: 15 (métrica de la plataforma, **no certificación**).
- Dificultad de aprendizaje: interpretar instrucciones extensas en inglés. Método útil: traducción corta, prueba propia, lectura de evidencias y preguntas de repaso.

## 1. Linux: conectividad y navegación
- **AttackBox**: estación desde la que operamos. **Lab Machine**: objetivo desplegado por la sala. Confirmar `hostname` y `whoami` antes de ejecutar comandos.
- `ssh usuario@IP`: sesión remota cifrada (SSH, puerto habitual TCP 22). Al primer acceso verificar identidad/huella del servidor por vía confiable. La contraseña no se muestra al escribirla.
- Prompt `usuario@host:~$` no se copia como parte del comando. `~` es home, `/` raíz, `.` directorio actual y `..` padre.
- `pwd`: ruta actual; `cd /var`: ir a /var; `cd ~`: volver a home; `ls`: listar; `ls -a`: incluir ocultos; `ls -la`: detalles/permisos y ocultos.
- `ls --help` muestra opciones; `man ls` abre manual si está instalado y funcionando.
- `touch` crear archivo vacío; `mkdir` crear carpeta; `cp origen destino` copiar; `mv` mover/renombrar; `rm` eliminar (con cuidado).
- Directorios: `/etc` configuración; `/var/log` registros; `/root` home del superusuario; `/tmp` temporales; `/home` hogares de usuarios.
- Error real: `cd note Documents/` falla porque `cd` no copia y se entregaron dos argumentos. Correcto: `cp note Documents/`. Un archivo llamado `.flag.txt` **no** es el mismo que `flag.txt`.

## 2. Usuarios, grupos y permisos
Ejemplo: `-rw-r----- 1 user2 user2 ... important`.
- Primer carácter: tipo de elemento (`-` archivo, `d` directorio).
- Tres bloques `rw-` (propietario), `r--` (grupo), `---` (otros).
- `r` leer, `w` escribir, `x` ejecutar; en directorios, `x` permite atravesar/entrar.
- `su -l user2`: cambio de usuario mediante autenticación válida; `whoami`: identidad efectiva de la sesión. `cat archivo`: mostrar contenido si existe permiso.
- Evidencia de sala: usuario inicial no podía leer `important` por permisos ordinarios; con cuenta `user2` se obtuvo `THM{…}` sin modificar el modo del archivo.
- `sudo` requiere autorización específica, no basta conocer la contraseña. Un error `not in the sudoers file` demuestra denegación de elevación.

## 3. Transferencias: wget, scp y servidor temporal
- `wget URL`: descargar por HTTP/HTTPS.
- `scp origen usuario@IP:/ruta/destino`: copiar hacia servidor por SSH; invertir origen/destino para descargar del remoto.
- En **Lab Machine**, desde el directorio a compartir: `python3 -m http.server 8000`. Sirve archivos por HTTP; por defecto no autentica y puede exponer archivos del directorio.
- En **AttackBox**, desde otra terminal: `wget http://IP_DEL_LAB:8000/.flag.txt`; comprobar con `ls -la`, `cat .flag.txt`.
- Incidente de aprendizaje: inicialmente se levantó el HTTP server en AttackBox, pero se intentó descargar desde Lab Machine → `Connection refused` (el servicio no estaba en ese destino).
- Evidencia correcta: servidor registró `GET /.flag.txt HTTP/1.1` con estado `200` e IP del cliente; `wget` devolvió `200 OK` y 20 bytes guardados. Flag `THM{…}`.
- Terminar servidor temporal con `Ctrl+C`. En producción no publicar información sensible mediante un servidor de prueba.

## 4. Procesos, PID y servicios
- `ps`: procesos ligados a la sesión; `ps aux`: vista amplia con USER, PID, %CPU, %MEM, STAT y COMMAND.
- PID identifica un proceso mientras existe; no asumir que siempre será el número anterior +1 en un sistema real. En la pregunta didáctica de la sala, 300 → 301.
- `ps aux | grep '[T]HM{'`: búsqueda de cadenas en argumentos visibles, evitando emparejar al propio grep. Solo usar en el sistema autorizado. En sala: `THM{…}`.
- Estados `R` ejecutándose/listo; `S` dormido; `T` detenido; `Z` zombie.
- `comando &` inicia trabajo en segundo plano; `Ctrl+Z` suspende; `jobs` enumera trabajos de **esa shell**; `bg` reanuda en segundo plano; `fg` trae al frente; `Ctrl+C` envía SIGINT normalmente.
- `SIGTERM` (15) solicita cierre ordenado; `SIGKILL` (9) fuerza terminación.
- `systemctl stop myservice`: detener ahora; `systemctl start myservice`: iniciar ahora; `systemctl enable myservice`: habilitar inicio automático (no es idéntico a iniciar ahora).

## 5. Cron: automatización y persistencia
- Cron ejecuta trabajos en un calendario. Formato clásico: `MIN HOUR DOM MON DOW COMMAND`.
- Ejemplo `0 */12 * * * comando`: cada día a las 00:00 y 12:00, según hora del sistema.
- `crontab -l` (letra ele, **no** número uno): listar trabajos del usuario; `crontab -e`: editar.
- Evidencia de sala: `@reboot /var/opt/processes.sh`. `@reboot` indica ejecución al iniciar el daemon cron, habitualmente en el arranque del sistema.
- En SOC revisar cron inesperado, script apuntado, dueño, permisos, contenido e historial de cambios. No toda tarea automática es maliciosa.

## 6. Paquetes y repositorios
- APT administra software en distribuciones como Ubuntu; `apt update` actualiza índices, **no instala** actualizaciones.
- `apt upgrade` actualiza paquetes; `apt install paquete`, `apt remove paquete`, `apt list --installed`.
- Fuentes habituales: `/etc/apt/sources.list` y `/etc/apt/sources.list.d/` (en versiones nuevas también fuentes deb822). La firma ayuda a verificar integridad/autenticidad de metadatos; no implica que cualquier paquete sea seguro.
- El tutorial muestra `apt-key`, método **obsoleto**; para nuevas configuraciones se prefieren claves acotadas con `signed-by`. No se instalaron repositorios externos en la sala.

## 7. Registros Apache: investigar evidencia HTTP
- `/var/log/apache2/access.log` solicitudes; `error.log` errores. Archivos rotados como `access.log.1` pueden preservar evidencia previa.
- En laboratorio `cat access.log` → Permission denied y `sudo` → no autorizado. Se revisó `ls -lh` y se detectó `access.log.1` legible.
- Evidencia observada:
  `10.9.232.111 - - [04/May/2021:18:18:16 +0000] "GET /catsanddogs.jpg HTTP/1.1" 200 51395`
- IP cliente: `10.9.232.111`; recurso: `catsanddogs.jpg`; método `GET`; estado `200`; bytes de respuesta `51395` según formato del log.
- Hallazgo de permisos en **entorno de práctica**: `access.log.1` figuraba con `-rwxrwxrwx` (777): permisos excesivos que podrían comprometer integridad del log. No afirmar explotación; recomendar revisión.
- No confundir logs de ejemplo del texto de la sala con registros efectivamente consultados.

## 8. Active Directory Basics
- AD DS: servicio de directorio del dominio. DC: controlador de dominio que proporciona servicios como autenticación y acceso al directorio.
- Objetos: usuarios (personas o identidades de servicio), equipos (ejemplo `TOM-PC$`), grupos, impresoras, etc.
- Grupo `Domain Admins`: administración privilegiada del dominio.
- **OU (Organizational Unit)**: organiza objetos, delega administración y facilita vincular GPO. **Grupo de seguridad**: agrupa principales para conceder permisos, y puede utilizarse para filtrar GPO.
- Un objeto usuario/equipo está ubicado en un contenedor/OU dentro del dominio, pero puede pertenecer a varios grupos.
- En sala `THM.local` se observaron OU IT, Management, Marketing, Research and Development y Sales; en IT, usuarios Claire, Mary, Phillip.
- Para nuevo departamento Quality Assurance, contenedor correcto: **Organizational Units**.
- `dsa.msc` abre Active Directory Users and Computers en Windows con herramientas administrativas instaladas.
- Relacionar con laboratorio personal Windows Server / GPO y separar evidencias de práctica de infraestructura real.

## 9. Diccionario inglés técnico
- access = acceso; permissions = permisos; owner = propietario; group = grupo.
- directory = directorio; file = archivo; hidden = oculto; source = origen; destination = destino.
- deploy = desplegar; instance = máquina desplegada; credentials = credenciales; remotely = remotamente.
- process = proceso; foreground = primer plano; background = segundo plano; stopped = suspendido/detenido; done = finalizado.
- boot = arranque; schedule = programar; service = servicio; logs = registros; request = solicitud.
- domain = dominio; domain controller = controlador de dominio; organizational unit = unidad organizativa; security group = grupo de seguridad.
- `Which...?` = ¿cuál...?; `What file did they access?` = ¿a qué archivo accedieron?; `When will it run?` = ¿cuándo se ejecutará?

## 10. Errores valiosos y siguientes pasos
1. Comprobar siempre **en qué equipo** se está trabajando (AttackBox frente a Lab Machine), especialmente después de reconectar.
2. No copiar el prompt `$ / #` ni confundir `crontab -l` con `-1`.
3. Para investigar permisos: primero `ls -l`; no alterar el archivo ni intentar saltarse controles.
4. Para logs rotados, validar su fecha, dueño, permisos y pertinencia; nunca asumir que `access.log` contiene todos los eventos.
5. Próxima sesión: concluir Active Directory Basics; luego practicar detección de inicios de sesión, grupos privilegiados y GPO dentro del laboratorio propio.

## Evidencia y alcance
Apuntes reconstruidos de capturas, terminales y respuestas confirmadas durante la sesión. No son un volcado literal de todas las pantallas. Los IP de laboratorios temporales fueron omitidos salvo la IP cliente requerida como evidencia de Apache. Los flags de capacitación son públicos de esa sala; no son credenciales reales. No subir passwords, claves, tokens, datos institucionales o capturas con información privada.
