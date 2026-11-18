# Reconocimiento de Active Directory con cuenta estándar (LDAP)

## Objetivo del laboratorio

Aprender, desde una cuenta de dominio de bajo privilegio, qué información de Active Directory puede consultar un atacante autenticado y cómo interpretar esa información desde el punto de vista ofensivo y defensivo.

Este bloque NO consiste en memorizar comandos. La lógica es:

1. Identificar servicios del dominio.
2. Confirmar qué servicio permite responder la pregunta que tenemos.
3. Consultar Active Directory con una cuenta estándar.
4. Entender qué devuelve AD.
5. Relacionar usuarios, grupos, equipos y posibles privilegios.
6. Dejar preparado el camino para futuras prácticas controladas de ataque + detección.

---

## Contexto del laboratorio

- Dominio: `bluelab.test`
- DC: `LAB-DC01`
- IP DC: `172.31.250.10`
- Cliente: `LAB-CLIENT01`
- IP cliente: `172.31.250.20`
- Kali: `172.31.250.40`
- Cuenta estándar usada para reconocimiento: `BLUELAB\victima`
- Cuenta administrativa descubierta: `adm-mxnx`
- Grupo administrativo de estaciones: `GG-WKS-ADMINS`

---

# 1. Reconocimiento inicial del Domain Controller

Desde Kali se consultaron los puertos típicos de Active Directory:

```bash
nmap -n -Pn -p 53,88,135,139,389,445,464,636,3268 172.31.250.10
```

### Qué significa cada puerto relevante

| Puerto | Servicio | Por qué importa |
|---|---|---|
| 53 | DNS | Resolución de nombres del dominio |
| 88 | Kerberos | Autenticación en AD |
| 135 | RPC | Llamadas remotas de Windows |
| 139 | NetBIOS | Servicios heredados Windows |
| 389 | LDAP | Consultas al directorio |
| 445 | SMB | Compartidos / autenticación SMB |
| 464 | Kerberos password change | Cambio de contraseña Kerberos |
| 636 | LDAPS | LDAP cifrado con TLS |
| 3268 | Global Catalog | Búsquedas globales del bosque |

### Lectura práctica

Si aparecen juntos 53 + 88 + 389 + 445 + 3268, es una señal muy fuerte de que estamos frente a un Domain Controller.

---

# 2. LDAP: concepto que hay que recordar

**LDAP = Lightweight Directory Access Protocol**

En este laboratorio:

- Active Directory = el directorio.
- LDAP = el protocolo para preguntarle cosas al directorio.
- `ldapsearch` = la herramienta usada desde Kali para hacer esas consultas.

Modelo mental:

```text
Active Directory = directorio
LDAP = idioma/protocolo de consulta
ldapsearch = herramienta para preguntar
```

---

# 3. Primer intento fallido y por qué falló

El primer `ldapsearch` quedó mal interpretado. La salida mostraba algo como:

```text
base <> (default)
filter: (objectclass=*)
requesting: -D victima@bluelab.test -b DC=bluelab,DC=test ...
```

Eso fue la pista.

Las opciones `-D`, `-b`, etc. no estaban siendo interpretadas como opciones de conexión/búsqueda, sino como atributos solicitados.

El DC respondió:

```text
In order to perform this operation a successful bind must be completed
```

### Enseñanza

Cuando algo falla, no asumir inmediatamente "LDAP está malo".

Primero revisar:

1. ¿Llegamos al servicio?
2. ¿La autenticación se hizo?
3. ¿La sintaxis fue interpretada correctamente?
4. ¿La base de búsqueda está correcta?
5. ¿El filtro LDAP está correcto?

---

# 4. LDAPS y validación de certificados

Se probó también LDAPS:

```bash
LDAPTLS_REQCERT=never ldapsearch ...
```

La idea de:

```bash
LDAPTLS_REQCERT=never
```

era decirle al cliente LDAP que, **solo para este laboratorio**, no rechazara la conexión por no confiar en el certificado TLS del servidor.

Esto NO:

- busca usuarios;
- autentica;
- entrega permisos;
- rompe LDAP.

Solo modifica la validación del certificado.

En este caso, LDAPS devolvió:

```text
Can't contact LDAP server (-1)
```

Por lo tanto, la solución final fue volver a LDAP por 389, que ya sabíamos que respondía.

---

# 5. Consulta correcta de usuarios por LDAP

Consulta utilizada:

```bash
ldapsearch -x -H ldap://172.31.250.10 -D 'victima@bluelab.test' -W -b 'DC=bluelab,DC=test' '(objectClass=user)' sAMAccountName
```

### Traducción humana

- `ldapsearch` → consultar LDAP.
- `-x` → autenticación simple.
- `-H ldap://172.31.250.10` → servidor LDAP objetivo.
- `-D 'victima@bluelab.test'` → identidad usada para autenticarse.
- `-W` → pedir contraseña de forma interactiva.
- `-b 'DC=bluelab,DC=test'` → raíz desde donde buscar.
- `(objectClass=user)` → buscar objetos clase user.
- `sAMAccountName` → mostrar nombre de inicio de sesión.

### Resultado observado

La cuenta estándar pudo descubrir objetos como:

- Administrador
- Invitado
- krbtgt
- LAB-DC01$
- LAB-CLIENT01$
- mxnx
- adm-mxnx
- victima

### Hallazgo importante

`objectClass=user` también devuelve equipos, porque en Active Directory los computadores heredan de la clase user.

Por eso aparecen cuentas terminadas en `$`:

```text
LAB-DC01$
LAB-CLIENT01$
```

Esas son cuentas de máquina.

---

# 6. Filtrar usuarios humanos

Consulta:

```bash
ldapsearch -x -H ldap://172.31.250.10 -D 'victima@bluelab.test' -W -b 'DC=bluelab,DC=test' '(&(objectCategory=person)(objectClass=user))' sAMAccountName
```

### Lógica del filtro LDAP

```text
&
= AND

objectCategory=person
AND
objectClass=user
```

La idea fue aproximarnos mejor a usuarios humanos y eliminar cuentas de equipo del resultado.

### Resultado relevante

Se identificaron usuarios como:

- Administrador
- Invitado
- krbtgt
- mxnx
- adm-mxnx
- victima

---

# 7. Enumeración de grupos

Consulta:

```bash
ldapsearch -x -H ldap://172.31.250.10 -D 'victima@bluelab.test' -W -b 'DC=bluelab,DC=test' '(objectClass=group)' cn
```

### Qué pedimos

- Objetos que sean grupos.
- Mostrar el atributo `cn`.

### Grupos observados

Entre muchos otros:

- Administradores
- Usuarios
- Invitados
- Operadores de copia de seguridad
- Usuarios de escritorio remoto
- Administradores de Hyper-V
- Controladores de dominio
- Administradores de esquema
- Administradores de empresas
- Admins. del dominio
- Usuarios del dominio
- Operadores de cuentas
- Acceso compatible con versiones anteriores de Windows 2000

### Importante

Ver que un grupo existe NO significa que `victima` pertenezca a él.

Enumeración y privilegio son cosas distintas.

---

# 8. Cadenas largas en la salida LDAP

En algunos resultados aparecieron cadenas largas en campos como:

```text
dn::
cn::
```

No eran contraseñas.

En LDIF, cuando un valor contiene determinados caracteres especiales, Unicode, tildes u otras codificaciones, puede mostrarse en Base64.

### Regla práctica

- `dn:` → texto normal
- `dn::` → valor codificado (frecuentemente Base64)

No confundir esto con hashes o passwords.

---

# 9. Consultar membresías de una cuenta interesante

Al descubrir una cuenta llamada `adm-mxnx`, se consultó su atributo `memberOf`:

```bash
ldapsearch -x -H ldap://172.31.250.10 -D 'victima@bluelab.test' -W -b 'DC=bluelab,DC=test' '(sAMAccountName=adm-mxnx)' memberOf
```

### Pregunta real detrás del comando

> ¿A qué grupos pertenece adm-mxnx?

### Resultado

```text
memberOf: CN=GG-WKS-ADMINS,OU=GRUPOS,DC=bluelab,DC=test
```

### Interpretación

`adm-mxnx` pertenece a:

```text
GG-WKS-ADMINS
```

En este laboratorio, ese grupo se usa para dar administración local sobre estaciones de trabajo mediante GPO.

Esto NO significa Domain Admin.

Sí significa que, desde la perspectiva de un atacante, `adm-mxnx` es una cuenta mucho más interesante que un usuario estándar.

---

# 10. Consultar quién pertenece a GG-WKS-ADMINS

Consulta:

```bash
ldapsearch -x -H ldap://172.31.250.10 -D 'victima@bluelab.test' -W -b 'DC=bluelab,DC=test' '(&(objectClass=group)(cn=GG-WKS-ADMINS))' member
```

### Pregunta real

> ¿Quiénes son miembros de GG-WKS-ADMINS?

### Resultado

```text
member: CN=adm mxnx,OU=ADMINISTRADORES,OU=USUARIOS,OU=BLUELAB,DC=bluelab,DC=test
```

Por ahora el grupo tiene como miembro relevante a:

```text
adm-mxnx
```

---

# 11. DN - Distinguished Name

Ejemplo observado:

```text
CN=adm mxnx,OU=ADMINISTRADORES,OU=USUARIOS,OU=BLUELAB,DC=bluelab,DC=test
```

Eso es el **DN (Distinguished Name)**.

Se puede leer como la dirección completa del objeto dentro de Active Directory:

```text
CN = objeto
OU = unidad organizativa
DC = componente del dominio
```

---

# 12. Relación descubierta hasta este punto

```text
victima
   |
   |-- cuenta estándar válida
   |
   |-- consulta LDAP
   |
   |-- descubre usuarios
   |
   |-- descubre grupos
   |
   |-- descubre adm-mxnx
   |
   |-- consulta memberOf
   |
   '-- descubre GG-WKS-ADMINS
```

Este es el comienzo de un mapa de relaciones de privilegios.

La idea ofensiva no es "ya tengo control".

La idea correcta es:

> Ya sé qué identidades pueden ser más interesantes y qué relaciones existen.

---

# 13. SPN - Service Principal Name

Luego se buscaron objetos con SPN:

```bash
ldapsearch -x -H ldap://172.31.250.10 -D 'victima@bluelab.test' -W -b 'DC=bluelab,DC=test' '(&(objectCategory=person)(objectClass=user)(servicePrincipalName=*))' sAMAccountName servicePrincipalName
```

### Qué significa SPN

**SPN = Service Principal Name**

Es un identificador usado por Kerberos para asociar un servicio con una identidad.

Ejemplo conceptual:

```text
MSSQLSvc/sql01.bluelab.test:1433
```

podría representar un servicio SQL registrado bajo una determinada cuenta.

---

# 14. Resultado real de la búsqueda de SPN

La consulta mostró principalmente:

- `LAB-DC01$`
- `LAB-CLIENT01$`
- `krbtgt`

Con SPN como:

- LDAP
- DNS
- HOST
- GC
- RPC
- RestrictedKrbHost
- kadmin/changepw

### Interpretación

No apareció una cuenta típica de servicio de usuario como:

```text
svc_sql
svc_backup
sqlservice
websvc
```

Por lo tanto, **no se encontró todavía una cuenta de servicio de usuario claramente candidata para Kerberoasting**.

---

# 15. Corrección del filtro para futuras búsquedas

Para evitar ruido y centrarse en usuarios de servicio:

```bash
ldapsearch -x -H ldap://172.31.250.10 -D 'victima@bluelab.test' -W -b 'DC=bluelab,DC=test' '(&(objectCategory=person)(objectClass=user)(servicePrincipalName=*)(!(sAMAccountName=krbtgt)))' sAMAccountName servicePrincipalName
```

Lógica:

```text
persona
AND
usuario
AND
tiene SPN
AND
no es krbtgt
```

---

# 16. Qué aprendimos realmente

No memorizar comandos completos.

Recordar el proceso mental:

```text
¿Qué quiero saber?
      |
¿Qué servicio responde esa pregunta?
      |
¿Qué protocolo usa?
      |
¿Qué herramienta puedo utilizar?
      |
¿Qué filtro necesito?
      |
¿Qué atributo quiero ver?
      |
¿Cómo interpreto el resultado?
```

Ejemplo:

```text
Quiero saber usuarios
→ Active Directory
→ LDAP
→ ldapsearch
→ filtro user
→ sAMAccountName
```

Otro:

```text
Quiero saber qué privilegios puede tener adm-mxnx
→ Active Directory
→ LDAP
→ busco sAMAccountName=adm-mxnx
→ pido memberOf
→ descubro GG-WKS-ADMINS
```

---

# 17. Valor Blue Team / SOC

Esta práctica sirve para entender qué puede hacer un atacante después de comprometer credenciales válidas de bajo privilegio.

El analista SOC debe poder distinguir:

- autenticación válida;
- enumeración;
- acceso rechazado;
- descubrimiento de usuarios/grupos;
- uso de LDAP;
- consultas de Kerberos;
- escalamiento real de privilegios.

También refuerza que:

```text
credencial válida != administrador
servicio visible != acceso autorizado
grupo visible != pertenencia
SPN visible != compromiso
```

---

# 18. Próxima etapa (NO ejecutada todavía)

Diseñar deliberadamente una cuenta de servicio débil dentro de .BLUELAB, por ejemplo:

```text
svc_sql
```

con un SPN de laboratorio, para estudiar de forma controlada:

1. creación de cuenta de servicio;
2. asignación de SPN;
3. descubrimiento desde `victima`;
4. solicitud de ticket Kerberos;
5. análisis de evidencias;
6. eventos del DC;
7. Wazuh;
8. explicación ofensiva + defensiva;
9. documentación para portafolio.

El objetivo es mantener un orden: primero comprender la estructura y luego romper deliberadamente el laboratorio para aprender detección y respuesta.

---

## Estado del laboratorio al cerrar esta etapa

- Reconocimiento del DC completado.
- LDAP funcional desde Kali.
- Cuenta `victima` validada como cuenta estándar útil para reconocimiento.
- Enumeración de usuarios completada.
- Enumeración de grupos completada.
- Relación `adm-mxnx -> GG-WKS-ADMINS` comprobada.
- SPN revisados.
- No se identificó todavía una cuenta de servicio de usuario claramente Kerberoasteable.
- Próximo bloque: crear una debilidad intencional y atacarla de forma controlada.
