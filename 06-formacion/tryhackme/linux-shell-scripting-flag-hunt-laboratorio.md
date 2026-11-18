# TryHackMe — Linux Shell Scripting: búsqueda de indicadores en logs (flag_hunt.sh)

**Fecha de práctica:** lunes 16 de noviembre de 2026  
**Ruta de estudio:** TryHackMe → Linux Shell Scripting → Task 6 «Practical Exercise»  
**Orientación:** Linux + Blue Team / SOC N1  
**Estado:** ✅ dos respuestas aceptadas por TryHackMe; script ejecutado en la VM del laboratorio.

> **Propósito de esta guía:** poder reproducir el ejercicio meses después, sabiendo qué se buscaba, por qué falló varias veces, qué corregimos, qué significan los comandos y cómo aplicar el aprendizaje a una investigación defensiva. Los ejemplos de ampliación están etiquetados como propuestos: no confundirlos con pruebas realmente ejecutadas.

## 1. El desafío original (lo que decía la sala)

TryHackMe había dejado un script Bash en `/home/user` de una máquina Ubuntu temporal. El script debía **buscar una palabra clave dentro de todos los archivos terminados en `.log` de una carpeta dada**. Había que completar **comillas dobles vacías** en el código antes de ejecutarlo.

Valores entregados por TryHackMe:

- **Keyword / palabra clave:** `thm-flag01-script`.
- **Directorio de búsqueda:** `/var/log`.
- **Primera pregunta:** *Which file has the keyword?* — «¿Qué archivo contiene esa palabra?».
- **Segunda pregunta:** *Where is the cat sleeping?* — «¿Dónde duerme el gato?».

La sala recomendaba `sudo su` para leer logs protegidos. En nuestro caso la elevación **falló por autenticación**, pero logramos terminar con el usuario normal porque el archivo que necesitábamos sí era legible. Esto enseña a separar claramente **error de ruta**, **error de permisos** y **error de sintaxis**.

**No guardar contraseñas ni IP temporales de la sala como si fueran valores permanentes.**

## 2. Conexión e inspección: primer paso antes de editar

Una vez conectado por SSH como usuario `user`, el prompt era similar a:

~~~text
user@tryhackme:~$
~~~

La máquina temporaria usó la IP `10.65.164.112` en esa sesión. En otras sesiones puede cambiar.

Para comprobar qué archivos teníamos:

~~~bash
ls -la /home/user
~~~

Encontramos realmente un archivo:

~~~text
-rwxr-xr-x 1 user user 463 Sep 23 2024 flag_hunt.sh
~~~

### Interpretación de esa línea

| Fragmento | Significado |
|---|---|
| `-` inicial | Archivo regular |
| `rwx` | Propietario: leer, escribir y ejecutar |
| `r-x` | Grupo: leer y ejecutar |
| `r-x` | Otros: leer y ejecutar |
| `user user` | Propietario y grupo |
| `463` | Tamaño en bytes en la captura |
| `flag_hunt.sh` | Nombre del script, extensión convencional Bash Shell |

**Decisión:** como el usuario era propietario y tenía permiso de escritura, **no necesitábamos root para editar el script**. Sí podrían faltar permisos para **leer otros archivos de `/var/log`**.

Luego leímos el script sin ejecutarlo:

~~~bash
cat /home/user/flag_hunt.sh
~~~

**`cat`** imprime el contenido del archivo. Esta separación importa: **leer no es ejecutar**. El script podría contener cambios peligrosos, así que primero se inspecciona.

## 3. Reconstrucción comentada del script

El script final, representando la lógica que vimos en las capturas, es:

~~~bash
#!/bin/bash

# Carpeta donde están los registros que queremos inspeccionar
directory="/var/log"

# Cadena exacta que buscamos (sin espacio final)
flag="thm-flag01-script"

echo "Flag search in directory: $directory in progress..."

# Recorre los archivos terminados en .log en esa carpeta
for file in "$directory"/*.log; do

    # grep -q busca en silencio; devuelve 0 si encuentra coincidencia
    if grep -q "$flag" "$file"; then

        # basename retira la parte de directorio de la ruta
        echo "Flag found in: $(basename "$file")"
    fi
done
~~~

> Nota metodológica: es la **reconstrucción limpia** de la lógica inspeccionada y corregida; no es una copia byte a byte del script del laboratorio. Se preservaron los comandos clave (`for`, `grep -q`, `basename`, `echo`) y la configuración observada.

### Explicación línea por línea

**`#!/bin/bash` (shebang):** solicita interpretar el archivo con Bash cuando se ejecuta directamente (`./flag_hunt.sh`); si lo iniciamos expresamente con `bash archivo.sh`, elegimos Bash desde el comando.

**`directory="/var/log"`:** crea/asigna una variable llamada `directory`, cuyo valor es el directorio con registros Linux. Sin espacios alrededor del `=`: en Bash `directory = "/var/log"` **no es una asignación válida**. Tampoco deben quedar espacios ocultos al final de la ruta.

**`flag="thm-flag01-script"`:** variable con la palabra a buscar. El texto es sensible a coincidencia exacta y a espacios adicionales, porque luego se utiliza con `grep`.

**`echo "Flag search ... $directory ..."`:** imprime avance; `$directory` se sustituye por el valor de la variable.

**`for file in "$directory"/*.log; do`:**
- `for` inicia un **bucle**.
- `file` es la variable temporal para el elemento actual.
- `in` indica los elementos a recorrer.
- `"$directory"` se expande a la carpeta **protegiendo los espacios dentro de su nombre**.
- `/*.log` añade barra separadora y comodín: **todos los nombres que acaban en `.log`**.
- `do` inicia el bloque de instrucciones del bucle.
- `done` lo termina.

**Punto técnico importante:** las comillas rodean **solo** la variable `"$directory"`, no todo `"$directory/*.log"`. Así, el comodín `*` queda **sin comillas y puede expandirse** a nombres de archivos. Si el directorio tiene espacios, la variable entre comillas sigue siendo un único componente de ruta.

**`if grep -q "$flag" "$file"; then`:**
- `grep` busca líneas que coinciden con el patrón.
- `-q` (*quiet*) no imprime líneas; se usa para saber **si hay al menos una coincidencia**.
- `"$flag"` evita separaciones involuntarias al expandir la palabra.
- `"$file"` protege nombres de archivo con espacios.
- `if` evalúa el código de salida de `grep`: **0 = encontró coincidencia**, **1 = no encontró**, **2 u otro error = fallo**.
- `then` introduce lo que se ejecutará si el resultado del comando es considerado exitoso.
- `fi` cierra el condicional.

**`echo "Flag found in: $(basename "$file")"`:**
- `basename` extrae el **nombre del archivo**, quitando el directorio.
- `$(...)` es **sustitución de comando** (*command substitution*): ejecuta lo de dentro y pega su salida en el texto.
- `echo` imprime el resultado.

### Ejemplo conceptual para entender el bucle

Si `/var/log` tiene `auth.log`, `system.log` y `authentication.log`, el bucle haría:

1. `file=/var/log/auth.log` → `grep -q` busca el texto.
2. `file=/var/log/system.log` → vuelve a buscarlo.
3. `file=/var/log/authentication.log` → si está presente, imprime `Flag found in: authentication.log`.

**No** significa que los tres archivos existieran o fueran legibles en nuestra VM; es solo una explicación de cómo funciona la iteración.

## 4. Cronología de los errores reales y por qué ocurrieron

Este fue el aprendizaje más valioso: **no basta con copiar un script; interpretamos su comportamiento y aislamos errores**.

### 4.1 Problema de `sudo`: autenticación rechazada

Probamos `sudo su` y `sudo -v`, pero el sistema devolvió:

~~~text
Sorry, try again.
sudo: 3 incorrect password attempts
~~~

Eso indica **autenticación no aceptada**; no es un mensaje de que `bash`, `grep` o la ruta de los logs estén mal.

Había dificultades adicionales con el teclado del AttackBox (distribución diferente; `@` y caracteres especiales). Intentamos comprobar la entrada y pegar texto, pero **no se obtuvo root durante la práctica**.

**Decisión correcta que permitió avanzar:** inspeccionar y editar `flag_hunt.sh` con los permisos del propietario; ejecutar la búsqueda como `user` primero. No insistir indefinidamente con contraseñas en una máquina de aprendizaje.

### 4.2 Primer fallo: buscaba `/*.log` en lugar de `/var/log/*.log`

Tras editar por primera vez, la ejecución mostró:

~~~text
Flag search in directory: /var/log in progress...
grep: /*.log: No such file or directory
~~~

**Interpretación:** el mensaje de estado mostraba la variable `directory`, pero **eso no demostraba que el bucle la usara correctamente**. El bucle estaba armado con una ruta incorrecta/comillas vacías.

Solución: revisar `for file in ...`, no solamente `echo`.

### 4.3 Segundo fallo: espacio al final de la variable o en la ruta

La consola pasó a imprimir errores parecidos a:

~~~text
grep: /var/log /*.log: No such file or directory
~~~

Una ruta con **un espacio después de `log`** no es la carpeta `/var/log`. En una de las capturas se observó un espacio final dentro de `directory`, y en otra la construcción del bucle también contenía un espacio antes de `/*.log`.

**Detalle sutil:** en Bash `"texto "` conserva el espacio al final. Aunque visualmente cueste detectarlo, `/var/log ` y `/var/log` son dos cadenas distintas.

Las líneas correctas quedaron:

~~~bash
directory="/var/log"
flag="thm-flag01-script"
for file in "$directory"/*.log; do
~~~

El espacio final en `flag` también haría fallar una búsqueda literal de la palabra sin ese espacio (salvo que el archivo lo contuviera exactamente).

**Diagnóstico reutilizable:** si la ruta del error tiene espacios imprevistos, revisar variables y comillas antes de culpar permisos o herramientas.

### 4.4 Tercer resultado: ¡encontramos la palabra, con otros logs denegados!

Finalmente se ejecutó:

~~~bash
bash /home/user/flag_hunt.sh
~~~

Y la captura mostró:

~~~text
Flag search in directory: /var/log in progress...
grep: /var/log/apport.log: Permission denied
grep: /var/log/auth.log: Permission denied
Flag found in: authentication.log
grep: /var/log/cloud-init-output.log: Permission denied
grep: /var/log/cloud-init.log: Permission denied
grep: /var/log/kern.log: Permission denied
~~~

**Conclusión precisa:**

- El bucle ya recorría la carpeta correcta.
- **`authentication.log` sí contenía la palabra**.
- No podíamos afirmar que los otros archivos no contenían la palabra: **no se pudieron leer**.
- Por eso no era válido decir «no hay coincidencias en el resto de `/var/log`». La cobertura era parcial debido a permisos.

**Primera respuesta aceptada por TryHackMe:** `authentication.log`.

### 4.5 Segundo hallazgo: extraer el contexto y no solo el nombre

Una vez encontrado `authentication.log`, en lugar de leer cientos de líneas, consultamos directamente la coincidencia y las líneas vecinas:

~~~bash
grep -n -C 3 "thm-flag01-script" /var/log/authentication.log
~~~

La salida real fue:

~~~text
/var/log/authentication.log
1-the cat is sleeping under the table
2:thm-flag01-script
~~~

**Explicación de la salida:**

- **`-n`** (*line number*): agrega número de línea.
- **`-C 3`** (*context*): incluye hasta 3 líneas antes y hasta 3 después del texto buscado, cuando existen.
- **`1-`**: línea 1 que se muestra como contexto; el guion la distingue de la coincidencia.
- **`2:`**: línea 2 que coincide; los dos puntos la señalan.
- La frase en línea 1 dice literalmente: `the cat is sleeping under the table`.

**Segunda respuesta aceptada por TryHackMe:** `under the table`.

✅ **Task 6:** ambas respuestas en verde; práctica terminada sin disponer de root.

---

## 5. Comandos operativos resumidos (qué buscamos y por qué)

| Comando | Qué hace | Motivo concreto en esta práctica |
|---|---|---|
| `ls -la /home/user` | Lista archivos, incluidos ocultos, con permisos | Descubrir el nombre del script y si podíamos editarlo |
| `cat /home/user/flag_hunt.sh` | Muestra el texto del script | Inspeccionar código **antes** de ejecutar |
| `nano /home/user/flag_hunt.sh` | Abre editor de texto | Completar valores y corregir espacios |
| `bash -n /home/user/flag_hunt.sh` | Revisa sintaxis sin ejecutar comandos | Separar problemas de sintaxis de errores de ejecución |
| `bash /home/user/flag_hunt.sh` | Ejecuta con Bash | Buscar palabra clave en logs |
| `grep -q "$flag" "$file"` | Busca coincidencia en silencio; evalúa estado | Condición del `if` |
| `basename "$file"` | Extrae nombre sin directorio | Mostrar `authentication.log` |
| `grep -n -C 3 ...` | Busca con números de línea y contexto | Encontrar la pista `under the table` |

### Diferencia entre leer, buscar, buscar en silencio y revisar sintaxis

~~~bash
# Leer todo el archivo (puede ser largo):
cat /var/log/authentication.log

# Mostrar líneas que coinciden:
grep "thm-flag01-script" /var/log/authentication.log

# Buscar e indicar número de línea y contexto:
grep -n -C 3 "thm-flag01-script" /var/log/authentication.log

# Solo comprobar coincidencia dentro de un if (sin imprimirla):
grep -q "thm-flag01-script" /var/log/authentication.log

# Verificar sintaxis del script (no ejecuta el bucle):
bash -n /home/user/flag_hunt.sh
~~~

**Diferencia entre `/var/log` y `/var/log/`:** las dos se refieren a la misma carpeta cuando no hay espacios añadidos; el problema observado era `/var/log ` (espacio **antes** del cierre de comillas) o un espacio intercalado entre `"$directory"` y `/*.log`.

**No confundir una validación `bash -n` exitosa con una ejecución sin errores:** la sintaxis puede estar bien y la ruta o permisos seguir fallando.

---

## 6. Conexión con SOC N1: IOC, logs y límites de una búsqueda

Esta tarea no era un incidente real; era un desafío con datos fabricados por TryHackMe. Pero ensayó una técnica útil de **triage**:

1. **Hipótesis:** una cadena específica podría aparecer en los registros.
2. **Fuente:** archivos `.log` de un directorio.
3. **Recopilación:** script recorre archivos.
4. **Detección:** `grep` busca coincidencias.
5. **Evidencia:** nombre de archivo y contenido de línea relevante.
6. **Corroboración:** contexto circundante para entender el significado.
7. **Limitación:** errores `Permission denied` = archivos **no examinados**, no resultado negativo.
8. **Conclusión:** «La cadena apareció en `authentication.log`; la línea anterior daba la respuesta del gato».

En un caso real, la cadena buscada podría ser una IP, dominio, nombre de usuario o mensaje de error. Una coincidencia **no significa automáticamente intrusión**. Revisar fuente del log, momento, host y contexto.

### Ejercicio de ampliación propuesto (no ejecutado en esta sala)

Encontrar fallos SSH en un archivo Linux existente:

~~~bash
grep -n -i "Failed password" /var/log/auth.log
~~~

Si no hay permiso, reportar ese límite y solicitar acceso autorizado; **no cambiar permisos ni eliminar protecciones a ciegas**.

### Variación defensiva mejorada: solo logs legibles, sin errores ruidosos

**Este bloque es una mejora de estudio, no el script original de TryHackMe.** A diferencia del original, avisa qué archivos no pudo leer:

~~~bash
#!/usr/bin/env bash

directory="/var/log"
pattern="thm-flag01-script"

for file in "$directory"/*.log; do
    # Si el comodín no corresponde a ningún archivo real, ignorarlo
    [ -f "$file" ] || continue

    if [ ! -r "$file" ]; then
        printf '[SIN PERMISO] %s\n' "$file" >&2
        continue
    fi

    if grep -Fq -- "$pattern" "$file"; then
        printf '[COINCIDENCIA] %s\n' "$file"
    fi
done
~~~

**Nuevas piezas explicadas:**
- `[ -f "$file" ]`: valida archivo regular.
- `continue`: salta al próximo elemento del bucle.
- `[ -r "$file" ]`: comprueba si aparentemente es legible para el usuario; algunos fallos aún pueden producirse por otras razones.
- `-F` (*fixed strings*): trata el patrón como texto literal y no como expresión regular.
- `--`: termina opciones, útil si el patrón comienza con guiones.
- `>&2`: envía advertencia a stderr, separando errores del resultado normal.
- `printf`: impresión controlada y predecible.

**Precaución SOC:** evitar modificar logs durante la investigación. El ejemplo solamente lee archivos. En producción conviene preservar evidencia, autorización y contexto temporal.

---

## 7. Preguntas para interiorizar (repetición espaciada)

**Ronda 1: recordar comandos**
1. ¿Qué muestra `ls -la /home/user` que no veíamos con `ls` normal?
2. ¿Qué significa `rwxr-xr-x`?
3. ¿Qué hace `cat` y qué NO hace?
4. ¿Qué hace `bash -n`?
5. ¿Qué hace `grep -q`?
6. ¿Qué hace `basename`?
7. ¿Qué significa `$(basename "$file")`?
8. ¿Qué representa `$directory`?

**Ronda 2: razonar errores**
9. ¿Por qué `/var/log /*.log` no encuentra nuestros registros?
10. ¿Qué diferencia hay entre `"abc"` y `"abc "` en Bash?
11. ¿Por qué una salida de `echo` con `/var/log` no garantiza que el bucle use bien esa ruta?
12. ¿Un `Permission denied` significa «el archivo no contiene la palabra»?
13. ¿Si `sudo` falla, necesariamente el script no puede funcionar?
14. ¿Qué diferencia hay entre corregir errores de sintaxis, errores de ruta y errores de permisos?

**Ronda 3: aplicar a SOC**
15. ¿Qué comando usarías para ver la coincidencia con 3 líneas alrededor?
16. ¿Qué diferencia hay entre `grep` y `grep -q`?
17. ¿Qué registrarías como evidencia junto al nombre del archivo?
18. ¿Cómo comunicarías el límite de visibilidad por logs inaccesibles?
19. ¿Por qué encontrar una IP en un log no prueba ataque?
20. ¿Qué parte de un script Bash hace las veces de bucle y qué parte hace condición?

### Respuestas breves (consultar después de intentar)
1. Archivos ocultos, propietarios y permisos.
2. Propietario lectura/escritura/ejecución; grupo y otros lectura/ejecución.
3. Muestra contenido; no lo ejecuta.
4. Comprueba sintaxis, no ejecuta búsqueda.
5. Retorno de coincidencia en silencio.
6. Quita directorio de una ruta.
7. Ejecuta basename y usa su salida dentro de echo.
8. Variable con directorio.
9. Espacio cambia la ruta.
10. El segundo contiene un espacio final.
11. echo y for pueden usar variables/rutas distintas.
12. No, es archivo no inspeccionado.
13. No: depende de permisos del script y archivos.
14. Analizar mensajes de error y causas distintas.
15. `grep -n -C 3 "patron" archivo.log`.
16. grep imprime coincidencias; -q solo retorna estado.
17. Hora, host, ruta, línea, contexto y restricciones.
18. «Cobertura parcial; ciertos logs no fueron leídos».
19. Puede ser actividad legítima, insuficiente para atribución.
20. `for ... do ... done` e `if ... then ... fi`.

---

## 8. Resultado final verificable

| Pregunta | Respuesta aceptada | Evidencia |
|---|---|---|
| Which file has the keyword? | **`authentication.log`** | `Flag found in: authentication.log` |
| Where is the cat sleeping? | **`under the table`** | `grep -n -C 3` mostró `1-the cat is sleeping under the table` |

**Status del laboratorio:** Task 6 ambas preguntas correctas. Task 7 «Conclusion» aún aparecía pendiente/plegada en la última captura.  
**No afirmar:** root obtenido, todos los logs examinados, limpieza de errores de permisos, ataque detectado o sistema comprometido.

### Mini resumen de memoria

`ls -la` **encuentra script** → `cat` **permite leerlo** → `nano` **permite editarlo** → `bash -n` **comprueba sintaxis** → `bash` **lo ejecuta** → `for` **recorre archivos** → `grep -q` **busca palabra** → `basename` **muestra el nombre** → `grep -n -C 3` **da evidencia contextual**.

**Aprendizaje central:** errores de permisos, espacios y rutas son problemas distintos. En investigación defensiva, una fuente inaccesible no es un resultado negativo; hay que declarar el límite de cobertura.
