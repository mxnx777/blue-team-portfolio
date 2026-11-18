# Guía práctica — OU, usuarios, grupos, GPO y mínimo privilegio

**Fecha de cierre de esta etapa:** 2026-07-30  
**Laboratorio:** `bluelab.test`  
**Controlador de dominio:** `LAB-DC01`  
**Cliente Windows 11:** `LAB-CLIENT01`  
**Objetivo:** organizar Active Directory, crear cuentas separadas por función y delegar administración local sobre estaciones de trabajo sin entregar privilegios de administrador del dominio.

---

## 1. Resultado conseguido

Se implementó una estructura básica de Active Directory con separación entre equipos, usuarios y grupos. También se creó una GPO que agrega un grupo del dominio al grupo local `Administradores` de las estaciones de trabajo.

Resultado final:

```text
BLUELAB\adm-mxnx
        ↓ miembro de
BLUELAB\GG-WKS-ADMINS
        ↓ agregado por GPO a
LAB-CLIENT01\Administradores
```

Con esto:

```text
BLUELAB\adm-mxnx
✅ puede administrar LAB-CLIENT01
❌ no es Domain Admin
❌ no administra LAB-DC01
❌ no modifica Active Directory por sí solo

BLUELAB\mxmx
✅ puede iniciar sesión en LAB-CLIENT01
✅ puede trabajar dentro de su perfil de usuario
❌ no pertenece al grupo local Administradores
❌ no puede modificar áreas protegidas sin elevación
```

> La cuenta diaria se creó como `BLUELAB\mxmx`. Se decidió conservar ese nombre porque ya está registrada y validada de esa forma.

---

## 2. Estructura de unidades organizativas

Se abrió la consola:

```text
dsa.msc
```

Nombre de la consola:

```text
Usuarios y equipos de Active Directory
```

Ruta gráfica:

```text
Administrador del servidor
→ Herramientas
→ Usuarios y equipos de Active Directory
```

Estructura creada:

```text
bluelab.test
└── BLUELAB
    ├── EQUIPOS
    │   └── WORKSTATIONS
    │       └── LAB-CLIENT01
    ├── USUARIOS
    │   ├── ESTANDAR
    │   │   └── mxmx
    │   └── ADMINISTRADORES
    │       └── adm-mxnx
    └── GRUPOS
        └── GG-WKS-ADMINS
```

### Qué función cumple cada OU

| OU | Función |
|---|---|
| `BLUELAB` | Raíz lógica para separar los objetos propios del laboratorio |
| `EQUIPOS` | Contenedor lógico para equipos administrados |
| `WORKSTATIONS` | Estaciones de trabajo Windows cliente |
| `USUARIOS` | Contenedor lógico para identidades del dominio |
| `ESTANDAR` | Cuentas de trabajo diario sin privilegios administrativos |
| `ADMINISTRADORES` | Cuentas separadas para tareas de administración |
| `GRUPOS` | Grupos de seguridad utilizados para asignar permisos |

### Cómo se creó cada OU

Para crear una OU se usó:

```text
Clic derecho sobre la ubicación padre
→ Nuevo
→ Unidad organizativa
```

Ejemplos:

```text
Clic derecho en bluelab.test
→ Nuevo
→ Unidad organizativa
→ BLUELAB
```

```text
Clic derecho en BLUELAB
→ Nuevo
→ Unidad organizativa
→ EQUIPOS
```

```text
Clic derecho en EQUIPOS
→ Nuevo
→ Unidad organizativa
→ WORKSTATIONS
```

Se dejó marcada:

```text
Proteger contenedor contra eliminación accidental
```

### Concepto aprendido

Una OU no entrega permisos por sí sola.

```text
OU = organiza objetos y define alcance de GPO
Grupo = reúne identidades y puede recibir permisos
GPO = aplica configuraciones
```

---

## 3. Movimiento de LAB-CLIENT01

Después de unir `LAB-CLIENT01` al dominio, el objeto apareció inicialmente dentro de:

```text
bluelab.test\Computers
```

Se movió a:

```text
bluelab.test\BLUELAB\EQUIPOS\WORKSTATIONS
```

### Ruta gráfica usada

```text
Usuarios y equipos de Active Directory
→ bluelab.test
→ Computers
→ clic derecho en LAB-CLIENT01
→ Mover
→ BLUELAB
→ EQUIPOS
→ WORKSTATIONS
→ Aceptar
```

### Por qué se movió

El contenedor predeterminado `Computers` no permite organizar de forma profesional el alcance de las GPO.

Al mover el equipo a `WORKSTATIONS` se consiguió:

- aplicar políticas solo a estaciones de trabajo;
- separar clientes de servidores;
- evitar afectar accidentalmente al controlador de dominio;
- administrar los equipos por función;
- preparar futuras políticas de hardening y auditoría.

Mover el objeto no cambió:

- nombre del equipo;
- dirección IP;
- pertenencia al dominio;
- contraseña de la cuenta de máquina;
- relación de confianza con el dominio.

Solo cambió su ubicación lógica dentro de Active Directory.

---

## 4. Creación de la cuenta diaria

Dentro de:

```text
BLUELAB\USUARIOS\ESTANDAR
```

se creó:

```text
BLUELAB\mxmx
```

### Ruta gráfica usada

```text
Usuarios y equipos de Active Directory
→ BLUELAB
→ USUARIOS
→ ESTANDAR
→ clic derecho
→ Nuevo
→ Usuario
```

Datos principales:

```text
Nombre de inicio de sesión: mxmx
SamAccountName: mxmx
UserPrincipalName: mxmx@bluelab.test
```

La cuenta quedó habilitada y como miembro de:

```text
Usuarios del dominio
```

Equivalente en inglés:

```text
Domain Users
```

### Cómo se revisó su grupo

```text
Clic derecho en mxmx
→ Propiedades
→ Miembro de
```

### Por qué es una cuenta de mínimo privilegio

No pertenece a grupos como:

```text
Administradores
Administradores del dominio
Enterprise Admins
```

Por eso puede trabajar normalmente, pero no administrar el sistema.

---

## 5. Creación de la cuenta administrativa separada

Dentro de:

```text
BLUELAB\USUARIOS\ADMINISTRADORES
```

se creó:

```text
BLUELAB\adm-mxnx
```

### Ruta gráfica usada

```text
Usuarios y equipos de Active Directory
→ BLUELAB
→ USUARIOS
→ ADMINISTRADORES
→ clic derecho
→ Nuevo
→ Usuario
```

Inicialmente también quedó solo como miembro de:

```text
Usuarios del dominio
```

Estar dentro de una OU llamada `ADMINISTRADORES` no la convirtió automáticamente en administradora.

### Concepto aprendido

```text
Nombre de la OU ≠ privilegio real
Nombre de un grupo ≠ privilegio real
Membresía + asignación efectiva = privilegio real
```

---

## 6. Creación del grupo de administración de estaciones

Dentro de:

```text
BLUELAB\GRUPOS
```

se creó:

```text
GG-WKS-ADMINS
```

### Ruta gráfica usada

```text
Usuarios y equipos de Active Directory
→ BLUELAB
→ GRUPOS
→ clic derecho
→ Nuevo
→ Grupo
```

Configuración:

```text
Nombre: GG-WKS-ADMINS
Ámbito: Global
Tipo: Seguridad
```

Significado del nombre:

```text
GG     = Grupo Global
WKS    = Workstations
ADMINS = Administradores
```

### Cómo se agregó adm-mxnx al grupo

```text
Abrir propiedades de GG-WKS-ADMINS
→ pestaña Miembros
→ Agregar
→ escribir adm-mxnx
→ Comprobar nombres
→ Aceptar
```

La relación quedó:

```text
BLUELAB\adm-mxnx
        ↓ miembro de
BLUELAB\GG-WKS-ADMINS
```

En este punto el grupo todavía no tenía permisos reales sobre el cliente. Solo agrupaba la identidad administrativa.

### Por qué seguía apareciendo “Usuarios del dominio”

`adm-mxnx` puede pertenecer a varios grupos simultáneamente:

```text
BLUELAB\adm-mxnx
├── Usuarios del dominio
└── GG-WKS-ADMINS
```

`Usuarios del dominio` es su grupo básico. `GG-WKS-ADMINS` es el grupo que después recibió el permiso administrativo local mediante GPO.

---

## 7. Creación y vínculo de la GPO

Se abrió la consola:

```text
gpmc.msc
```

Nombre:

```text
Administración de directivas de grupo
```

Ruta gráfica:

```text
Administrador del servidor
→ Herramientas
→ Administración de directivas de grupo
```

En la consola se expandió:

```text
Bosque: bluelab.test
└── Dominios
    └── bluelab.test
        └── BLUELAB
            └── EQUIPOS
                └── WORKSTATIONS
```

Sobre `WORKSTATIONS` se usó:

```text
Clic derecho
→ Crear un GPO en este dominio y vincularlo aquí...
```

Nombre creado:

```text
GPO-WKS-ADMINISTRADORES-LOCALES
```

Con esa acción se hicieron dos cosas:

```text
1. Se creó la GPO
2. Se vinculó a la OU WORKSTATIONS
```

### Concepto aprendido

Crear una GPO no basta. Para que se aplique debe estar:

```text
1. Creada
2. Configurada
3. Vinculada a una OU, dominio o sitio
```

---

## 8. Dónde está exactamente la configuración que entrega el permiso

Esta es la ruta que conviene practicar varias veces.

### Paso 1 — Abrir la consola correcta

En `LAB-DC01`:

```text
Administrador del servidor
→ Herramientas
→ Administración de directivas de grupo
```

No se abre el Panel de control normal de Windows.

### Paso 2 — Buscar la GPO vinculada

En el panel izquierdo:

```text
Bosque: bluelab.test
→ Dominios
→ bluelab.test
→ BLUELAB
→ EQUIPOS
→ WORKSTATIONS
→ GPO-WKS-ADMINISTRADORES-LOCALES
```

### Paso 3 — Abrir el editor

```text
Clic derecho sobre GPO-WKS-ADMINISTRADORES-LOCALES
→ Editar
```

### Paso 4 — Seguir la ruta dentro del editor

```text
Configuración del equipo
└── Preferencias
    └── Configuración del Panel de control
        └── Usuarios y grupos locales
```

> `Configuración del Panel de control` es el nombre de una carpeta dentro del editor de GPO. No significa abrir el Panel de control del escritorio de Windows.

### Paso 5 — Ver la regla creada

En el panel derecho aparece el elemento que modifica:

```text
Administradores (integrados)
```

Al abrir sus propiedades se ve:

```text
Acción: Actualizar
Grupo local: Administradores (integrados)
Miembro agregado: BLUELAB\GG-WKS-ADMINS
Acción del miembro: ADD
```

Se dejaron desmarcadas:

```text
Eliminar todos los usuarios miembros
Eliminar todos los grupos miembros
```

### Por qué se usó Actualizar

La acción `Actualizar` agrega el grupo del dominio sin eliminar los miembros existentes del grupo local `Administradores`.

Esto evita borrar por error:

- la cuenta local integrada `Administrador`;
- otros administradores ya existentes;
- grupos administrativos agregados previamente.

### Cadena completa del permiso

```text
BLUELAB\adm-mxnx
        ↓ miembro de
BLUELAB\GG-WKS-ADMINS
        ↓ la GPO agrega ese grupo a
LAB-CLIENT01\Administradores
        ↓ por lo tanto
adm-mxnx administra LAB-CLIENT01
```

La OU organiza.  
El grupo reúne usuarios.  
La GPO coloca el grupo dentro del grupo local correcto.  
El grupo local `Administradores` entrega el privilegio.

---

## 9. Aplicación de la GPO en LAB-CLIENT01

En `LAB-CLIENT01` se abrió PowerShell como administrador y se ejecutó:

```powershell
gpupdate /force
```

Resultado:

```text
La actualización de la directiva de equipo se completó correctamente.
Se completó correctamente la actualización de directiva de usuario.
```

### Concepto aprendido

`gpupdate /force` confirma que Windows procesó las políticas, pero no demuestra por sí solo que una configuración específica haya quedado aplicada.

Por eso fue necesario verificar el resultado real.

---

## 10. Verificación del grupo local Administradores

En `LAB-CLIENT01` se ejecutó:

```powershell
Get-LocalGroupMember -Group "Administradores"
```

Resultado relevante:

```text
BLUELAB\GG-WKS-ADMINS    ActiveDirectory
```

También aparecieron otros miembros:

```text
BLUELAB\Admins. del dominio
LAB-CLIENT01\Administrador
LAB-CLIENT01\LAB-CLIENT01
```

### Diferencia de origen

| PrincipalSource | Significado |
|---|---|
| `ActiveDirectory` | Cuenta o grupo perteneciente al dominio |
| `Local` | Cuenta propia de `LAB-CLIENT01` |

### Evidencia final

La aparición de:

```text
BLUELAB\GG-WKS-ADMINS
```

dentro del grupo local `Administradores` demostró que la GPO funcionó correctamente.

---

## 11. Inicio de sesión con la cuenta estándar

Se inició sesión en `LAB-CLIENT01` usando:

```text
BLUELAB\mxmx
```

Se comprobó la identidad con:

```powershell
whoami
```

Resultado:

```text
bluelab\mxmx
```

Esto confirmó:

- autenticación con una cuenta del dominio;
- sesión abierta dentro de `LAB-CLIENT01`;
- funcionamiento de Active Directory;
- credenciales correctas.

---

## 12. Verificación de mínimo privilegio

Se ejecutó:

```powershell
whoami /groups | findstr "S-1-5-32-544"
```

El SID:

```text
S-1-5-32-544
```

corresponde al grupo local integrado:

```text
Administradores
```

Con `BLUELAB\mxmx`, el comando no devolvió resultados.

Interpretación:

```text
BLUELAB\mxmx
→ no pertenece al grupo local Administradores
→ no tiene privilegios administrativos en la sesión
```

---

## 13. Prueba práctica de permisos

Con `BLUELAB\mxmx` se intentó crear o modificar contenido dentro de:

```text
C:\Program Files
```

Windows solicitó permisos administrativos.

Resultado:

```text
✅ la cuenta puede escribir dentro de su perfil
❌ la cuenta no puede modificar libremente áreas protegidas
```

Rutas normales de escritura:

```text
C:\Users\mxmx\Desktop
C:\Users\mxmx\Documents
C:\Users\mxmx\Downloads
```

Rutas protegidas:

```text
C:\Windows
C:\Program Files
Registro HKLM
Configuraciones críticas del sistema
```

### Concepto aprendido

Ser usuario estándar no significa no tener permiso de escritura.

```text
Permiso de escritura = depende de permisos NTFS
Ser administrador = permite elevar privilegios y administrar el sistema
```

---

## 14. Inicio de sesión y elevación con adm-mxnx

Se inició sesión en `LAB-CLIENT01` con:

```text
BLUELAB\adm-mxnx
```

Se abrió PowerShell como administrador y se comprobó:

```powershell
whoami
```

Resultado:

```text
bluelab\adm-mxnx
```

Después:

```powershell
whoami /groups | findstr "S-1-5-32-544"
```

Resultado relevante:

```text
BUILTIN\Administradores
S-1-5-32-544
```

Esto confirmó:

```text
✅ adm-mxnx es administrador local de LAB-CLIENT01
✅ puede elevar mediante UAC
❌ no es Administrador del dominio
```

### UAC y token administrativo

Aunque una cuenta pertenezca a `Administradores`, Windows puede iniciar el escritorio con un token limitado.

```text
PowerShell normal
→ sesión sin elevar

PowerShell como administrador
→ token administrativo activo
```

---

## 15. Diferencias entre las cuentas usadas

| Cuenta | Origen | Uso correcto |
|---|---|---|
| `LAB-CLIENT01\Administrador` | Local | Recuperación o administración local del cliente |
| `BLUELAB\mxmx` | Dominio | Trabajo diario con mínimo privilegio |
| `BLUELAB\adm-mxnx` | Dominio | Administración local delegada en workstations |
| `BLUELAB\Administrador` | Dominio | Administración crítica del dominio |

### Regla operativa

```text
BLUELAB\mxmx
→ navegar, trabajar, abrir archivos, tareas diarias

BLUELAB\adm-mxnx
→ elevar privilegios cuando sea necesario administrar el cliente

BLUELAB\Administrador
→ administración crítica de Active Directory
```

---

## 16. Errores y aprendizajes durante la práctica

### Error 1 — Confusión entre mxmx y mxnx

La cuenta fue creada como:

```text
mxmx
```

pero se intentó iniciar sesión usando:

```text
mxnx
```

La verificación correcta fue:

```powershell
Get-ADUser -Filter * |
Select-Object Name,SamAccountName,UserPrincipalName,Enabled
```

Resultado:

```text
mxmx   mxmx   mxmx@bluelab.test   True
```

Aprendizaje:

> Antes de restablecer contraseñas o investigar bloqueos, comprobar siempre el `SamAccountName` real.

### Error 2 — whomai en lugar de whoami

```text
Incorrecto: whomai
Correcto:   whoami
```

### Error 3 — Confundir procesamiento con resultado

```text
gpupdate /force
```

no era evidencia final de que el grupo hubiese sido agregado.

La evidencia final fue:

```powershell
Get-LocalGroupMember -Group "Administradores"
```

### Error 4 — Confundir “Configuración del Panel de control”

No se debía abrir el Panel de control normal de Windows.

La ubicación correcta está dentro del editor de GPO:

```text
Configuración del equipo
→ Preferencias
→ Configuración del Panel de control
→ Usuarios y grupos locales
```

---

## 17. Conceptos profesionales aprendidos

### Mínimo privilegio

Una cuenta recibe solo los permisos necesarios para cumplir su función.

### Separación de cuentas

La cuenta diaria y la cuenta administrativa deben ser distintas.

### Administración delegada

Una cuenta puede administrar estaciones de trabajo sin convertirse en administradora del dominio.

### GPO

```text
GPO = conjunto de configuraciones centralizadas
OU = alcance lógico de aplicación
Grupo = agrupación de identidades y asignación de permisos
```

### Herencia de permisos por grupo

`adm-mxnx` no aparece directamente dentro de `Administradores` porque el permiso se entrega a través del grupo:

```text
adm-mxnx
→ GG-WKS-ADMINS
→ Administradores locales
```

Esto facilita agregar o quitar administradores sin editar cada equipo individualmente.

---

## 18. Prueba práctica para repetir todo sin mirar la solución

La idea es repetir el flujo para que quede aprendido, no memorizado de una sola vez.

### Preparación

Antes de practicar:

1. apagar las VMs;
2. crear un snapshot de `LAB-DC01` y `LAB-CLIENT01`;
3. encender ambas VMs;
4. confirmar que el cliente sigue en el dominio.

### Objetivo de la prueba

Crear una segunda cadena temporal:

```text
BLUELAB\adm-prueba
        ↓ miembro de
BLUELAB\GG-WKS-ADMINS-PRUEBA
        ↓ agregado por una GPO temporal a
LAB-CLIENT01\Administradores
```

### Parte A — Active Directory

Sin mirar la solución completa:

- crear `adm-prueba` dentro de `USUARIOS\ADMINISTRADORES`;
- crear `GG-WKS-ADMINS-PRUEBA` dentro de `GRUPOS`;
- configurarlo como Global y Seguridad;
- agregar `adm-prueba` como miembro del grupo.

### Parte B — GPO

- abrir Administración de directivas de grupo;
- crear `GPO-WKS-ADMINS-PRUEBA` vinculada a `WORKSTATIONS`;
- editarla;
- llegar a `Usuarios y grupos locales`;
- usar acción `Actualizar`;
- modificar `Administradores (integrados)`;
- agregar `BLUELAB\GG-WKS-ADMINS-PRUEBA`;
- no marcar las opciones de eliminación de miembros.

### Parte C — Validación

En `LAB-CLIENT01`:

```powershell
gpupdate /force
Get-LocalGroupMember -Group "Administradores"
```

Después iniciar sesión con:

```text
BLUELAB\adm-prueba
```

Validar:

```powershell
whoami
whoami /groups | findstr "S-1-5-32-544"
```

### Limpieza después de la prueba

Opción segura:

```text
Revertir los snapshots
```

Opción manual:

1. quitar el vínculo o eliminar la GPO temporal;
2. eliminar `GG-WKS-ADMINS-PRUEBA`;
3. eliminar `adm-prueba`;
4. ejecutar `gpupdate /force` en el cliente;
5. verificar que el grupo temporal desapareció de Administradores.

---

## 19. Checklist de memoria

Antes de mirar la respuesta, intentar completar mentalmente:

```text
¿Dónde creo OU y usuarios?
→ Usuarios y equipos de Active Directory

¿Dónde creo y vinculo GPO?
→ Administración de directivas de grupo

¿Dónde está la regla de administradores locales?
→ Configuración del equipo
→ Preferencias
→ Configuración del Panel de control
→ Usuarios y grupos locales

¿Qué entrega realmente el privilegio?
→ El grupo local Administradores

¿Qué conecta al grupo del dominio con el grupo local?
→ La GPO

¿Qué demuestra que la GPO se procesó?
→ gpupdate /force

¿Qué demuestra que el cambio quedó aplicado?
→ Get-LocalGroupMember -Group "Administradores"
```

---

## 20. Estado actual de esta etapa

| Control | Estado |
|---|---|
| OU raíz `BLUELAB` | Correcto |
| OU de equipos y workstations | Correcto |
| `LAB-CLIENT01` dentro de `WORKSTATIONS` | Correcto |
| OU de usuarios estándar y administrativos | Correcto |
| Cuenta diaria `BLUELAB\mxmx` | Correcto |
| Cuenta administrativa `BLUELAB\adm-mxnx` | Correcto |
| Grupo `GG-WKS-ADMINS` | Correcto |
| `adm-mxnx` agregado al grupo | Correcto |
| GPO vinculada a `WORKSTATIONS` | Correcto |
| Grupo agregado a administradores locales | Correcto |
| Inicio de sesión con cuenta estándar | Correcto |
| Verificación de mínimo privilegio | Correcto |
| Prueba de acceso a carpeta protegida | Correcto |
| Inicio de sesión con `adm-mxnx` | Correcto |
| Elevación UAC con `adm-mxnx` | Correcto |
| SID de Administradores validado | Correcto |

---

## 21. Próximos pasos recomendados

1. Crear snapshots con el estado actual validado.
2. Repetir la prueba temporal de administración delegada.
3. Confirmar que `adm-mxnx` no puede administrar `LAB-DC01`.
4. Crear una GPO base de hardening para `WORKSTATIONS`.
5. Configurar auditoría avanzada.
6. Revisar eventos de inicio de sesión y elevación.
7. Guardar capturas como evidencia sin exponer contraseñas.

---

## 22. Cómo explicarlo en una entrevista

> Organicé un dominio Windows mediante unidades organizativas para separar usuarios, equipos y grupos. Creé una cuenta diaria de mínimo privilegio y una cuenta administrativa separada. Luego implementé un grupo global de seguridad y una GPO vinculada únicamente a la OU de estaciones de trabajo para delegar administración local sin entregar privilegios de Domain Admin. Configuré la GPO mediante Preferencias, Usuarios y grupos locales, usando la acción Actualizar para conservar los administradores existentes. Finalmente validé la aplicación con gpupdate, Get-LocalGroupMember, whoami, revisión del SID S-1-5-32-544, elevación UAC y una prueba real de acceso a una ruta protegida.