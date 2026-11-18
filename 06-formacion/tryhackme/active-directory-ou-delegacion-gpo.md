# Guía integral TryHackMe — Active Directory Basics (Tasks 1–9)

**Práctica documentada:** noviembre de 2026 · **Actualización:** 2026-11-10  
**Tipo:** laboratorio educativo de TryHackMe — Windows AD Basics  
**Estado:** se recorrió la teoría hasta Task 9 y se configuraron varias prácticas; falta probar aplicación efectiva de GPO y completar Auto Lock Screen.  
**Entorno:** dominio ficticio `thm.local`. No corresponde a infraestructura productiva.

> **Objetivo:** entender qué se configura, por qué, cómo se valida y qué riesgo representa. Los nombres e IP del laboratorio son temporales. No publicar contraseñas temporales ni datos de producción.

## 1. Modelo mental

| Elemento | Qué es | Para qué sirve |
|---|---|---|
| Active Directory (AD DS) | Directorio de identidades y objetos | Centralizar cuentas, equipos, grupos y autenticación |
| Domain Controller (DC) | Servidor que ejecuta AD DS | Gestionar el dominio y sus servicios |
| OU (Organizational Unit) | Contenedor lógico del dominio | Organizar objetos, delegar administración y aplicar GPO |
| Security Group | Conjunto de principales de seguridad | Dar acceso a recursos y filtrar políticas |
| GPO (Group Policy Object) | Colección de configuraciones | Establecer parámetros de seguridad y funcionamiento |
| GPMC (`gpmc.msc`) | Consola de políticas | Crear, administrar y vincular GPO |
| GPO Editor | Editor de configuración | Ajustar reglas dentro de cada GPO |
| SYSVOL | Recurso compartido del controlador de dominio | Distribuir archivos y plantillas de GPO |

**Diferencia esencial:** crear GPO ≠ configurar GPO ≠ vincular GPO ≠ verificar su aplicación. Una GPO creada pero sin vínculos no alcanza automáticamente a usuarios ni equipos.

## 2. Organización de usuarios y computadoras

### Usuarios / departamentos de ejemplo
- Management: Daniel.
- Marketing: Mark.
- Sales: Sophie y Thomas.
- IT: Phillip, Mary y Claire.

En la práctica apareció una OU adicional **Research and Development**, no incluida en el organigrama, con el usuario Bob. Se revisó antes de eliminarla. Para eliminar una OU protegida se habilitó **View > Advanced Features**, luego **Properties > Object > Protect object from accidental deletion** (desmarcar) y finalmente **Delete**. La confirmación de eliminación de subárbol advierte que puede borrar los objetos contenidos: en un dominio real siempre inventariar y autorizar previamente.

### Computadores
Se revisó el contenedor predeterminado `Computers` y se distinguieron 7 estaciones de trabajo:
- LPT-PHILLIP, LPT-SOPHIE, LPT-THOMAS.
- PC-CLAIRE, PC-DANIEL, PC-MARK, PC-MARY.

Y 3 servidores:
- SRV-DB01, SRV-DB02, SRV-WEB01.

TryHackMe propone dos OU separadas **Workstations** y **Servers** bajo `thm.local` y conservar los controladores de dominio en `Domain Controllers`. En nuestras capturas se crearon OU de estaciones/servidores **dentro de THM** (una quedó como `WORKSTATION`), una desviación respecto al diagrama que conviene corregir al repasar. Las respuestas conceptuales de la sala fueron **7** y **yay**. Importante: las respuestas aceptadas no prueban que todos los equipos se hayan movido.

**Razón:** servidores y estaciones requieren controles diferentes; las GPO se pueden vincular a sus propias OU.

## 3. Delegación: soporte técnico sin Domain Admin

**Objetivo:** permitir a Phillip restablecer contraseñas de usuarios en **Sales** sin privilegios de administrador total del dominio.

1. En `dsa.msc`, clic derecho en la OU `Sales` > **Delegate Control**.
2. **Add** > seleccionar `phillip` y confirmar con **Check Names**.
3. Marcar únicamente **Reset user passwords and force password change at next logon**.
4. **Next > Finish**.
5. Conectar por RDP al equipo de laboratorio como `THM\phillip` y comprobar `whoami`.
6. En PowerShell como Phillip:

```powershell
Set-ADAccountPassword sophie -Reset -NewPassword (Read-Host -AsSecureString -Prompt "New Password") -Verbose
Set-ADUser -Identity sophie -ChangePasswordAtLogon $true -Verbose
```

Se observaron mensajes `VERBOSE` para `CN=Sophie,OU=Sales,OU=THM,DC=thm,DC=local`, sin errores visibles. Finalmente se accedió al escritorio de Sophie y TryHackMe aceptó la flag: evidencia de realización del ejercicio.

**Incidencia de RDP:** Remmina mostró `Change user password before connecting` al exigir cambio de contraseña antes de completar el inicio de sesión. Para terminar **solo esta práctica** se desmarcó temporalmente la obligación mediante:

```powershell
Set-ADUser -Identity sophie -ChangePasswordAtLogon $false
```

**Advertencia:** esto reduce el control de seguridad. En producción se debe habilitar un flujo compatible para que el usuario cambie su clave, no usar la desactivación como solución permanente.

**Seguridad:** principio de mínimo privilegio. Una cuenta de soporte comprometida con derechos delegados puede afectar a los usuarios dentro del alcance de esa OU. Si la auditoría está habilitada, analizar eventos como **4724** (intento de restablecimiento de contraseña) y modificaciones de cuentas.

## 4. Group Policy: teoría operativa

`Group Policy Objects` almacena las GPO en la consola; las OU son los destinos donde se vinculan. El editor siempre organiza las reglas en:
- **Computer Configuration:** reglas que se procesan para el equipo.
- **User Configuration:** reglas que se procesan para el usuario.

Una GPO vinculada al dominio puede heredarse por las OU descendientes, según herencia, precedencia, permisos de seguridad, filtros WMI y condiciones de procesamiento. **No suponer** que mover objetos a otra OU anula políticas heredadas.

GPO observadas inicialmente: `Default Domain Policy`, `Default Domain Controllers Policy`, `RDP policy`.

### Política de contraseñas del dominio
En el ejercicio se cambió la longitud mínima de **7 a 10 caracteres** en:

```text
GPMC > Group Policy Objects > Default Domain Policy > Edit
Computer Configuration > Policies > Windows Settings
> Security Settings > Account Policies > Password Policy
> Minimum password length = 10
```

Se usó `gpupdate /force` en la máquina del ejercicio para solicitar nuevo procesamiento. Verificar antes de afirmar que el cambio es efectivo en todos los equipos o para todas las cuentas.

**Precaución:** `Default Domain Policy` se encuentra vinculada al dominio. No colocar en ella restricciones departamentales que deban excluir a IT.

### SYSVOL
Ubicación habitual en el DC:

```text
C:\Windows\SYSVOL\sysvol
\\thm.local\SYSVOL
```

SYSVOL distribuye componentes de GPO. Los clientes necesitan poder leer los recursos pertinentes, pero sus permisos de escritura deben estar estrictamente restringidos. Alteraciones no autorizadas de GPO/SYSVOL son un riesgo de seguridad.

Comandos útiles:

```powershell
gpupdate /force
gpresult /r
gpresult /h C:\Temp\reporte-gpo.html
```

`gpupdate` solicita actualizar políticas; `gpresult` ayuda a comprobar cuáles se aplicaron. Algunas requieren nuevo inicio de sesión o reinicio.

## 5. Caso realizado: restringir Panel de control fuera de IT

**Necesidad:** usuarios de Management, Marketing y Sales no deben acceder a Panel de control/Configuración; personal de IT debe conservar acceso.

### Paso A — crear objeto
1. Abrir `gpmc.msc`.
2. `Forest > Domains > thm.local > Group Policy Objects`.
3. Clic derecho > **New**.
4. Nombre exacto: `Restrict Control Panel Access`. Starter GPO: `(none)`.

### Paso B — configurar
1. Clic derecho sobre la GPO nueva > **Edit**.
2. `User Configuration > Policies > Administrative Templates > Control Panel`.
3. Abrir **Prohibit access to Control Panel and PC settings**.
4. Seleccionar **Enabled** y guardar.

### Paso C — vincular
1. En GPMC, expandir `thm.local > THM`.
2. Clic derecho en **Marketing** > **Link an Existing GPO** > seleccionar la nueva GPO.
3. Repetir en **Management** y **Sales**.
4. **No vincular a IT**.

**Evidencia observada:** en la consola aparecieron los vínculos con `Management`, `Marketing`, `Sales`; `Link Enabled = Yes`; en la OU Sales se veía el vínculo de la GPO. Esto demuestra **configuración y vinculación**, no todavía ejecución efectiva en una cuenta cliente.

### Error detectado y corregido
Inicialmente se entró a **Default Domain Policy** para configurar la restricción. Esa era la GPO equivocada: su alcance es el dominio. Después se creó la GPO dedicada y se activó allí la configuración. **Pendiente revisar que la restricción accidental en Default Domain Policy haya quedado en Not Configured.**

### Validación aún pendiente
- Confirmar `Scope > Links` con las tres OU y sin IT.
- Revisar `Default Domain Policy` para evitar una restricción heredada no deseada.
- Iniciar sesión como un usuario de Sales, actualizar políticas y comprobar que Panel de control está bloqueado.
- Iniciar sesión como usuario de IT y comprobar que no recibe ese bloqueo.
- Guardar resultados `gpresult` (redactando datos sensibles).

## 6. Próximo ejercicio, NO completado: Auto Lock Screen

Objetivo del material: bloqueo de pantalla tras **5 minutos = 300 segundos** de inactividad.

La guía propone una GPO separada `Auto Lock Screen`, con una configuración de **Computer Configuration**; podría vincularse al dominio para abarcar estaciones, servidores y controladores, evaluando herencia y alcance.

**Pendiente:** crear, configurar, vincular, forzar actualización y verificar. No registrar como realizado hasta tener resultados.

## 7. Checklist de repaso

- [x] Identificar DC, OU, grupos, objetos y cuentas de máquina.
- [x] Delegar restablecimiento de contraseña limitado a Sales y probar con Phillip.
- [x] Revisar SYSVOL y actualización de GPO en la teoría.
- [x] Cambiar longitud mínima a 10 en el laboratorio.
- [x] Crear `Restrict Control Panel Access` y habilitar la restricción.
- [x] Vincular a Management, Marketing y Sales; sin vínculo en IT.
- [ ] Auditar estado de `Default Domain Policy`.
- [ ] Probar aplicación efectiva con usuarios de Sales e IT.
- [ ] Completar Auto Lock Screen y verificar 300 segundos.
- [ ] Adjuntar capturas depuradas y resultados de políticas.

## 8. Autoevaluación (sin mirar soluciones)

1. ¿Cuál es la diferencia entre una OU y un grupo de seguridad?
2. ¿Por qué una GPO creada pero no vinculada todavía no restringe a Sales?
3. ¿Qué cambia entre User Configuration y Computer Configuration?
4. ¿Por qué **no** conviene poner la restricción de Panel de control en Default Domain Policy?
5. ¿Qué significa `Link Enabled: Yes` y qué **no** demuestra?
6. Si una GPO se vincula al dominio, ¿podría llegar a las OU hijas?
7. ¿Qué hace `gpupdate /force` y cómo verificarías el resultado?
8. ¿Qué riesgo representa dar Domain Admin a alguien que solo debe resetear contraseñas?
9. ¿Qué es SYSVOL y por qué sus permisos de escritura son críticos?
10. ¿Qué error operativo surgió con Remmina al forzar cambio de contraseña?

### Respuestas breves
1. OU organiza y permite políticas/delegación; grupo asigna permisos y puede filtrar aplicación.
2. Falta un vínculo con un ámbito aplicable.
3. El primero configura al usuario; el segundo al equipo.
4. Por alcance excesivo: podría restringir también IT y otros usuarios.
5. Vínculo activo; no garantiza procesamiento ni aplicación efectiva.
6. Sí, por herencia, según filtros y precedencias.
7. Fuerza reprocesamiento; verificar con `gpresult` y pruebas reales.
8. Privilegios innecesarios y mayor impacto si se compromete la cuenta.
9. Recurso compartido de archivos de políticas; escritura indebida permitiría manipulación.
10. NLA/cliente RDP no completó cambio de clave previo a la sesión.

---
**Metodología de portafolio:** observación → hipótesis → configuración → verificación → evidencia → pendientes. No confundir respuestas aceptadas por el curso con cambios efectivos sobre una infraestructura.


---

# Parte II. Autenticación, arquitectura de dominios y cierre de sala

Esta ampliación corresponde a **Task 7 (Authentication Methods)**, **Task 8 (Trees, Forests and Trusts)** y **Task 9 (Conclusion)**. Se lee después de las prácticas de AD/GPO anteriores. En vez de memorizar respuestas, reconstruye mentalmente qué dispositivo habla con cuál, qué dato intercambia y qué evento deja.

## 9. Diccionario de siglas — leer en voz alta en cada repaso

| Sigla / nombre | Nombre completo en inglés | En español | En una frase |
|---|---|---|---|
| AD | Active Directory | Directorio activo | Identidades, equipos, grupos y políticas |
| AD DS | Active Directory Domain Services | Servicios de dominio de Active Directory | Rol Windows Server que implementa el directorio |
| DC | Domain Controller | Controlador de dominio | Servidor que autentica y gestiona dominio |
| OU | Organizational Unit | Unidad organizativa | Organiza usuarios/equipos y permite delegar/GPO |
| GPO | Group Policy Object | Objeto de directiva de grupo | Configuraciones que se aplican a usuarios/equipos |
| GPMC | Group Policy Management Console | Consola de administración de directivas de grupo | Crea, edita y vincula GPO |
| DNS | Domain Name System | Sistema de nombres de dominio | Resuelve nombres; crítico para localizar DC y servicios |
| KDC | Key Distribution Center | Centro de distribución de claves | Servicio del DC que emite tickets Kerberos |
| AS | Authentication Service | Servicio de autenticación | Parte del KDC que emite TGT |
| TGT | Ticket-Granting Ticket | Ticket para solicitar tickets | Permite pedir tickets de servicios sin reiterar contraseña |
| TGS | Ticket-Granting Service | Servicio que concede tickets | Parte del KDC que expide tickets de servicio; en la sala también se usa «TGS» para el ticket |
| SPN | Service Principal Name | Nombre principal de servicio | Identifica servicio e identidad que lo ejecuta |
| SMB | Server Message Block | Protocolo de archivos compartidos | Acceso de red a recursos Windows |
| NTLM | NT LAN Manager | Mecanismo de autenticación Windows heredado | Desafío y respuesta; no enviar contraseña en claro |
| NLA | Network Level Authentication | Autenticación a nivel de red | Autentica antes de establecer sesión gráfica RDP |
| RDP | Remote Desktop Protocol | Protocolo de escritorio remoto | Acceso a escritorio remoto |
| SIEM | Security Information and Event Management | Gestión de información y eventos de seguridad | Centraliza y correlaciona logs |
| SOC | Security Operations Center | Centro de operaciones de seguridad | Monitorea, detecta, investiga incidentes |

**Regla de memoria:** TGT = «tengo autorización para PEDIR tickets»; ticket de servicio = «tengo un ticket para ESTE servicio». KDC = «quien entrega los tickets»; SPN = «cuál servicio estoy pidiendo».

## 10. Task 7 — Authentication Methods / Métodos de autenticación

### 10.1 Pregunta central

Cuando alguien inicia sesión como usuario de un dominio, ¿cómo demuestra a un servidor que ya está autenticado?

- **Kerberos:** protocolo preferido en dominios Windows modernos; usa tickets y claves de sesión.
- **NTLM / NetNTLM:** mecanismo heredado de desafío-respuesta que puede quedar habilitado por compatibilidad. No confundir el protocolo **NTLM** con **NetNTLM** (nombre común para intercambios de autenticación NTLM capturados).
- No se debe simplificar a «todas las contraseñas están almacenadas en el DC»: AD DS almacena verificadores/secretos, como hashes y claves, no una lista de contraseñas de todos los usuarios legible en texto claro.

### 10.2 Kerberos paso a paso: por qué existen DOS tickets

**Ejemplo:** un usuario Phillip inicia sesión en `thm.local` y quiere abrir una carpeta del servidor de archivos.

**Fase A — pedir un TGT (Ticket-Granting Ticket)**

1. El equipo localiza el KDC (Key Distribution Center), habitualmente alojado en un controlador de dominio (DC).
2. El cliente inicia una solicitud **AS-REQ** (Authentication Service Request). En el flujo clásico mostrado por TryHackMe demuestra conocimiento de un secreto mediante una marca temporal cifrada (**preautenticación**). Kerberos también admite otros mecanismos modernos de preautenticación.
3. El KDC valida la prueba y responde **AS-REP** (Authentication Service Reply) con un **TGT** y material de clave de sesión para el cliente.
4. El TGT se cifra/protege con la clave de la cuenta especial del dominio **krbtgt**; el cliente lo conserva, pero no necesita descifrar su contenido.

**Por qué:** el usuario no tiene que retransmitir la contraseña cada vez que accede a un recurso. El TGT funciona como credencial para solicitar tickets de servicios durante su vigencia.

**Fase B — pedir un ticket para un servicio concreto**

5. El cliente identifica el servicio mediante su **SPN (Service Principal Name)**; por ejemplo `MSSQLSvc/sql01.bluelab.test:1433` para un servicio SQL en un ejercicio de laboratorio.
6. Envía **TGS-REQ** (Ticket-Granting Service Request) al KDC con el TGT, un autenticador protegido y el SPN solicitado.
7. El KDC verifica el TGT y entrega **TGS-REP** (Ticket-Granting Service Reply), que incluye un **ticket de servicio** y material de clave de sesión cliente-servicio. La parte del ticket para el servidor está protegida con una clave de la cuenta que ejecuta ese servicio.

**Por qué:** los tickets de servicio tienen un destinatario específico; no son una contraseña universal para cualquier servidor.

**Fase C — presentar ticket al servidor**

8. El cliente envía al servidor el ticket de servicio y un autenticador (**AP-REQ**).
9. El servicio valida el ticket con su clave, revisa autenticador y demás condiciones; puede responder **AP-REP** en autenticación mutua.
10. La **autenticación** no equivale automáticamente a **autorización**: el servicio y los permisos (por ejemplo NTFS/SMB) determinan qué acciones puede realizar el usuario.

**Diagrama para memorizar**

```text
CLIENTE (usuario Phillip)            KDC / DC                 SERVIDOR
        |                               |                       |
        |------ AS-REQ ---------------->|                       |
        |<----- AS-REP: TGT ------------|                       |
        |                               |                       |
        |------ TGS-REQ (TGT + SPN) --->|                       |
        |<----- TGS-REP: service ticket-|                       |
        |                               |                       |
        |--------------------- AP-REQ: ticket ------------------>|
        |<-------------------- respuesta / acceso ---------------|
```

**Diferencia que suele confundirse:** `TGT` y `TGS` no son la misma cosa. Formalmente el TGS es el servicio emisor y el ticket para el recurso es el *service ticket*. TryHackMe llama TGS al ticket en su explicación inicial; conocer ambos usos permite leer documentación sin confundirse.

### 10.3 NetNTLM paso a paso: desafío-respuesta

**Ejemplo:** el mismo usuario intenta abrir una carpeta SMB cuando el flujo de autenticación utiliza NTLM.

1. **Authentication Request**: cliente solicita acceso al servidor.
2. **Challenge**: el servidor envía un desafío aleatorio, no su contraseña.
3. **Response**: cliente calcula respuesta criptográfica a partir de material asociado a su secreto, desafío y otros datos (NTLMv2).
4. **Verificación**: para una cuenta de dominio, servidor puede consultar a DC por el mecanismo correspondiente; el DC compara lo esperado.
5. **Allow/Deny**: el servidor comunica si autentica al usuario.
6. Incluso si la autenticación funciona, **los permisos del recurso** siguen decidiendo qué puede leer o modificar.

```text
CLIENTE                 SERVIDOR                  DC
  |--- solicitud ------->|                         |
  |<-- challenge --------|                         |
  |--- response -------->|--- validación --------->|
  |                      |<-- correcto / falló ----|
  |<-- resultado --------|                         |
```

**Matiz técnico indispensable:** la contraseña en texto claro normalmente **no viaja** en esta autenticación; no obstante, el intercambio puede ser abusado (por ejemplo, **NTLM relay**) si no existen mitigaciones adecuadas. Capturar una respuesta NTLM no equivale automáticamente a recuperar una contraseña.

### 10.4 Tabla comparativa de estudio

| Punto | Kerberos | NTLMv2 |
|---|---|---|
| Idea | Tickets emitidos por KDC | Desafío y respuesta |
| Emisor de tickets | KDC (en DC) | No emite TGT/TGS |
| Servicio designado | SPN / ticket de servicio | Servidor presenta desafío |
| Contraseña enviada en texto plano | No | No |
| Evidencia típica en DC | 4768, 4769, 4771 | 4776 (según auditoría/caso) |
| Riesgo de ejemplo | Kerberoasting si credenciales de servicio débiles | Relay si no se protegen protocolos |
| Puertos de referencia | Kerberos TCP/UDP 88 | NTLM no tiene «puerto NTLM» único; se transporta mediante SMB, HTTP, etc. |

### 10.5 Relación concreta con el laboratorio personal Blue Team

**No mezclar entornos:** la sala TryHackMe usó `thm.local`. El laboratorio personal tiene `.BLUELAB` (en ejemplos `bluelab.test`) y servidores/usuarios distintos.

El usuario relacionó correctamente la teoría con una práctica previa de **Kerberoasting** sobre una cuenta de servicio `svc_sql`, asociada al SPN `MSSQLSvc/sql01.bluelab.test:1433`. En ese ejercicio, desde un usuario con permisos ordinarios se solicitó material de ticket de servicio Kerberos y se estudió un formato como:

```text
$krb5tgs$23$*svc_sql$BLUELAB.TEST$...
```

**Qué significa ese texto:** `krb5tgs` identifica un formato de extracción de ticket de servicio Kerberos y `23` el tipo de cifrado Kerberos **RC4-HMAC** (etype 23), no un puerto ni un Event ID. El material puede utilizarse para probar contraseñas **fuera de línea** en un laboratorio autorizado. El objetivo defensivo es comprender el riesgo de cuentas de servicio con contraseñas débiles y configuraciones heredadas.

Después se analizó un acceso SMB usando las credenciales de `svc_sql`; se asoció a un evento **4624, Logon Type 3 (Network)** y un paquete de autenticación **NTLM**. Es importante distinguir:

- **Primero:** obtención de material asociado al ticket Kerberos.
- **Después:** uso posterior de credenciales en otro mecanismo (SMB/NTLM).
- **No confundir:** no es necesariamente *Pass-the-Ticket*; y **solicitar** un ticket no es por sí mismo una intrusión.

**Qué buscar en Windows:**

| ID de evento | Dónde mirarlo típicamente | Significado / pregunta del analista |
|---|---|---|
| 4768 | DC, Security | ¿Quién pidió un TGT? |
| 4769 | DC, Security | ¿Quién pidió un ticket de servicio?, ¿qué SPN/servicio? |
| 4771 | DC, Security | ¿Hubo fallo de preautenticación Kerberos? |
| 4776 | DC o equipo que valida | ¿Hubo validación NTLM? |
| 4624 | Equipo destino, Security | ¿Qué cuenta inició sesión correctamente? ¿Qué Logon Type? |
| 4625 | Equipo destino, Security | ¿Qué intento de inicio de sesión falló? |
| 4688 | Equipo con auditoría de procesos | ¿Qué proceso se creó? |

`Logon Type 2`: interactivo, en la consola.  
`Logon Type 3`: acceso por red, por ejemplo SMB.  
`Logon Type 10`: sesión remota interactiva, típicamente RDP.

**Triage SOC N1 de un 4769 sospechoso**:

1. Determinar ventana temporal, cuenta que solicita, DC y servicio solicitado.
2. Revisar IP/cliente (si el evento dispone de ella), SPN, tipo de cifrado y cantidad de solicitudes.
3. Comparar con patrón histórico y finalidad del servicio; un volumen alto no es automáticamente ataque.
4. Correlacionar con 4624/4625, validaciones NTLM y actividad de red en equipos destino.
5. Verificar si son cuentas de servicio privilegiadas o con controles débiles.
6. Documentar hipótesis, evidencias a favor y en contra, impacto y escalamiento. **No bloquear por intuición.**

### 10.6 Respuestas exactas Task 7 con explicación

1. **Will a current version of Windows use NetNTLM as the preferred authentication protocol by default?** → `nay`. En dominios modernos Kerberos es el preferido; NTLM es fallback/compatibilidad.
2. **What type of ticket allows us to request further tickets known as TGS?** → `Ticket Granting Ticket`. Es el **TGT** recibido del KDC.
3. **When using NetNTLM, is a user's password transmitted over the network at any point?** → `nay`. Se intercambia desafío y respuesta, no la contraseña en texto claro.

**Si se te olvida:** «TGT permite pedir tickets», «TGS permite llegar a servicios», «NTLM desafío-respuesta».

## 11. Task 8 — Trees, Forests and Trusts / Árboles, bosques y relaciones de confianza

### 11.1 Dominio, OU, Tree y Forest NO son sinónimos

- **Domain / Dominio:** frontera administrativa de identidades, cuentas y objetos bajo un dominio AD, por ejemplo `thm.local`.
- **OU (Organizational Unit):** contenedor **dentro de un dominio** para organizar objetos, delegar y vincular GPO. No crea dominio nuevo.
- **Tree / Árbol:** uno o más dominios que comparten un **espacio de nombres DNS contiguo**. Puede haber dominio padre e hijos.
- **Forest / Bosque:** una o varias estructuras de árboles AD que comparten configuración, esquema y relaciones forestales; pueden tener distintos nombres DNS.

**Mnemotecnia:** OU está **dentro** de dominio; dominio está **dentro** de árbol; árbol está **dentro** de bosque. Un bosque de un solo dominio también es posible.

### 11.2 Tree — un espacio de nombres

TryHackMe propone una empresa con oficinas UK y US:

```text
                   thm.local
                   DC-ROOT
                  /       \
       uk.thm.local       us.thm.local
           DC-UK              DC-US
```

Los tres dominios forman **un Tree**, porque `uk.thm.local` y `us.thm.local` son hijos DNS del mismo padre `thm.local`.

**Qué cambia respecto a tres OU**:

| Solución | Qué separa | Qué mantiene compartido |
|---|---|---|
| OU Marketing / Sales / IT | Organización y delegación **dentro de un dominio** | Misma base de dominio |
| Dominios hijos UK / US | Dominios con administración y políticas independientes | Pertenecen a la estructura de árbol/bosque |
| GPO vinculada a una OU | Alcance de directiva de grupo | No crea nueva frontera de dominio |

Cada dominio tiene sus cuentas y puede tener sus propios DC y GPO. El grupo **Domain Admins** suele administrar su dominio, no automáticamente el vecino.

**Enterprise Admins:** grupo de altos privilegios que existe en el dominio raíz del bosque y tiene capacidades administrativas de alcance forestal. Tratarlo como grupo crítico y limitar al máximo miembros y uso. Un bosque es una frontera de seguridad especialmente relevante; crear dominios hijos no garantiza aislamiento total ante compromiso de administradores forestales.

### 11.3 Forest — árboles distintos en un bosque

Supongamos que THM adquiere MHT; mantienen diferentes nombres DNS:

```text
FOREST
├── TREE THM
│   ├── thm.local
│   ├── uk.thm.local
│   └── us.thm.local
└── TREE MHT
    ├── mht.local
    ├── eu.mht.local
    └── asia.mht.local
```

**Tree** agrupa dominios en espacio de nombres contiguo; **Forest** puede reunir árboles con diferentes espacios de nombres. Todos los dominios dentro de un mismo bosque comparten elementos estructurales de AD; no es simplemente «dos empresas conectadas por VPN».

### 11.4 Trust Relationships — dirección que confunde

Una **Trust Relationship / Relación de confianza** permite que un dominio **reconozca identidades autenticadas en otro** bajo las condiciones establecidas. **No da por sí sola acceso a archivos**, porque todavía hacen falta permisos/autorizar recursos.

**One-way Trust (confianza unidireccional):**

Si **AAA confía en BBB**, entonces **usuarios de BBB pueden ser autorizados** a recursos de AAA.

```text
Confianza:   AAA ----------confía en----------> BBB
Acceso:      AAA <------ usuarios de BBB -------
             (recursos AAA)     (identidades BBB)
```

**Regla mental:** el **dominio que contiene el recurso confía** en el dominio de los usuarios. La dirección del vínculo de confianza y la de acceso son contrarias en ese ejemplo.

**Two-way Trust (confianza bidireccional):** ambas partes confían en la autenticación de las identidades de la otra. Sigue sin significar permisos automáticos.

Al agregar dominios hijos al mismo bosque normalmente se establecen relaciones bidireccionales transitivas por defecto; al integrar bosques separados, los tipos y condiciones de confianza dependen del diseño. No asumir que «todo está permitido».

### 11.5 Caso ejemplo: carpeta compartida entre dominios

**Escenario:** Ana pertenece a `uk.thm.local` y necesita leer `\\fileserver.asia.mht.local\reportes`.

Checklist conceptual:

1. Verificar conectividad, resolución DNS y rutas necesarias entre los dominios.
2. Comprobar que hay una **relación de confianza adecuada** entre los dominios/bosques implicados y que permite autenticar la identidad de Ana.
3. Autorizar a Ana, preferiblemente mediante grupos apropiados, en los permisos del recurso.
4. Revisar permisos de **share** y **NTFS**: ambos condicionan el acceso efectivo por SMB.
5. Probar con Ana y una identidad sin permisos; documentar éxito y rechazo.

**Seguridad:** confianza ≠ autorización; un usuario reconocido no debe tener acceso a todos los archivos.

### 11.6 Respuestas exactas Task 8

1. **What is a group of Windows domains that share the same namespace called?** → `Tree`.
2. **What should be configured between two domains for a user in Domain A to access a resource in Domain B?** → `A Trust Relationship`. **¡Incluir la A inicial!** Esta fue la diferencia entre la primera respuesta `Trust Relationship` y la cadena exacta que esperaba la plataforma.

## 12. Task 9 — Conclusion / Conclusión

El texto de la sala explica que **Active Directory Basics es introducción**, no equivale a desplegar ni proteger correctamente un AD productivo.

La Task 9 muestra **“Click and continue learning!”** y un campo **“No answer needed”**: presionar **Check**, sin escribir respuesta.

Los caminos mencionados:
- **Active Directory Hardening**: endurecer y defender AD, especialmente relevante para Blue Team.
- **Compromising Active Directory**: entender técnicas de abuso de errores de configuración, útil para generar y detectar evidencia en un entorno autorizado.

No se infiere haber completado otras salas ni certificado a partir de terminar esta.

## 13. Rutina de repetición espaciada y evaluación SOC

Usar estas preguntas en voz alta; responder **sin abrir la solución**, después explicar por qué. Repetirlas mezcladas, no solo en el orden de la sala. Revisar hoy, mañana, dentro de 3 días y en una semana.

### Bloque A — conceptos Active Directory
1. Explica en 30 segundos diferencias entre AD, DC, dominio y OU.
2. ¿Por qué un grupo de seguridad no equivale a una OU?
3. Enumera las cuatro etapas de una GPO: creación, edición, vínculo y comprobación.
4. ¿Por qué no colocar una restricción departamental en Default Domain Policy?
5. ¿Qué es SYSVOL y por qué importa controlar quién puede modificarlo?
6. ¿Qué se concedió exactamente en la delegación a Phillip y por qué no Domain Admin?

### Bloque B — autenticación
7. Expande en inglés/español KDC, TGT, TGS y SPN.
8. Relata en orden AS-REQ, AS-REP, TGS-REQ, TGS-REP, AP-REQ.
9. ¿Por qué un TGT no es lo mismo que un ticket para SQL?
10. ¿Por qué «no enviar contraseña» no vuelve a NTLM inmune a relay?
11. Si ves `$krb5tgs$23`, ¿qué representa 23?
12. ¿Qué evento Windows se relaciona con un ticket de servicio Kerberos?
13. ¿Qué evento y Logon Type esperarías en un acceso SMB exitoso por red?
14. ¿Cómo distingues una solicitud legítima de múltiples 4769 frente a sospecha de Kerberoasting?

### Bloque C — organización / confianza
15. ¿Cuál es la diferencia entre `Sales` como OU y `uk.thm.local` como dominio hijo?
16. ¿Qué es un Tree? Construye un ejemplo con dominio padre y dos hijos.
17. ¿Qué es un Forest? ¿Puede contener árboles con nombres DNS diferentes?
18. ¿Qué privilegios diferencian Domain Admins de Enterprise Admins?
19. Si AAA confía en BBB, ¿quién puede recibir permisos sobre recursos de quién?
20. ¿Una confianza de dos vías concede acceso automáticamente a todos los recursos?

**Autoevaluación orientativa:** si una respuesta no sale, no solo mirar la solución; reescribir el concepto, dar un ejemplo y repetirlo al final de la sesión. Alternar siempre práctica, teoría y eventos.

## 14. Evidencias y límites de lo que podemos afirmar

**Verificado con capturas de la sala de TryHackMe:**
- Editor de GPO y configuración de restricción de Panel de control.
- Creación y vínculos de `Restrict Control Panel Access` con Management, Marketing, Sales (IT excluido).
- Progreso por Task 7, Task 8 y visualización de conclusión Task 9.
- Respuesta exacta `A Trust Relationship` mostrada por interfaz.

**No verificado a partir de esas capturas:**
- Que una sesión real de Sales haya recibido la política correctamente.
- Que Default Domain Policy haya quedado totalmente limpia de la configuración previa.
- Que se haya finalizado **Auto Lock Screen** o comprobado en 300 segundos.
- Que la última Task 9 haya quedado marcada como aceptada tras presionar Check (la imagen mostraba el botón).

**Próxima práctica local propuesta (todavía no ejecutada):** en .BLUELAB generar una solicitud Kerberos legítima, inspeccionar 4768/4769 en DC, contrastar una sesión SMB 4624 Type 3 en destino y guardar capturas con fechas/usuarios anonimizados. Después documentar resultados reales, no resultados esperados.

---
**Lectura sugerida:** primero secciones 1–8 (AD/OU/GPO), luego 9–10 (Kerberos/NTLM), 11 (Tree/Forest/Trust) y finalmente 13 (autoevaluación). La guía es material de aprendizaje y portafolio, no procedimiento para cambiar producción sin evaluación y respaldo.
