# TryHackMe — Public Key Cryptography Basics

**Fecha:** 25 de noviembre de 2026 · **Ruta:** Cyber Security 101 → Public Key Cryptography Basics
**Propósito:** entender RSA, Diffie-Hellman, claves SSH, firmas digitales, certificados y GPG, y cómo se ven en un análisis SOC.

> Sin flags ni respuestas literales de la sala. Los valores numéricos de ejemplo son propios.
> Relacionado: [Cryptography Basics](cryptography-basics-guia-soc.md), [Networking Secure Protocols](networking-secure-protocols-guia-soc.md).

## 1. Diccionario de siglas

| Sigla | Nombre en inglés | Significado |
|---|---|---|
| RSA | Rivest–Shamir–Adleman | Algoritmo asimétrico basado en factorizar números grandes |
| DH | Diffie-Hellman | Intercambio de claves |
| ECC / ECDSA | Elliptic Curve Cryptography / ECC Digital Signature Algorithm | Curvas elípticas / firma con curvas elípticas |
| Ed25519 | — | Algoritmo de firma de curva elíptica usado en SSH |
| CA | Certificate Authority | Autoridad de certificación |
| PGP / GPG | Pretty Good Privacy / GNU Privacy Guard | Cifrado y firma de archivos y correos (estándar OpenPGP) |
| IV | Initialization Vector | Valor aleatorio que acompaña a un cifrado por bloques |
| CBC | Cipher Block Chaining | Modo de cifrado por bloques encadenados |

## 2. Cuatro propiedades

| Propiedad | Pregunta | Se apoya en |
|---|---|---|
| Authentication | ¿Con **quién** hablo? | Claves, certificados |
| Authenticity | ¿Este **mensaje** viene de esa persona? | Firma digital |
| Integrity | ¿Alguien lo **cambió**? | Hash + firma |
| Confidentiality | ¿Alguien más lo **lee**? | Cifrado |

Autenticación = la persona; autenticidad = el mensaje. El simétrico aporta sobre todo confidencialidad; el asimétrico suma las otras tres.

## 3. La regla de las claves (para no confundirse)

**Paso 1 — ¿Quiero OCULTAR o PROBAR?**

| Quiero... | Uso... | Analogía |
|---|---|---|
| **Ocultar** (que solo lo lea el destinatario) | **Pública del que recibe** | Le pongo su candado |
| **Probar que fui yo** (firmar) | **Mi privada** | Mi llave, que solo tengo yo |

**Paso 2 — Ancla práctica:** la clave SSH. La **pública** se copia al servidor (`~/.ssh/authorized_keys`); la **privada nunca sale del equipo** y lleva permisos 600. La privada nunca viaja.

Ejemplos:
- Alicia manda algo **secreto** a Bob → cifra con la **pública de Bob**.
- Bob quiere **probar** que firmó un documento → cifra el hash con **su privada**.

## 4. Intercambio de claves (cifrado híbrido)

El asimétrico es lento, así que se usa una sola vez para enviar la clave simétrica "dentro de una caja cerrada con el candado del servidor". Luego todo va con simétrico.

| Analogía | Sistema |
|---|---|
| Código secreto | Algoritmo y clave simétrica |
| Candado | Clave pública |
| Llave del candado | Clave privada |

## 5. RSA

Multiplicar dos primos grandes es fácil; factorizar el producto es inviable con primos de ~300 dígitos.

| Variable | Qué es |
|---|---|
| `p`, `q` | Primos secretos |
| `n = p × q` | Público |
| `φ(n) = (p − 1)(q − 1)` | Se usa para calcular `d` |
| `e` | Exponente público → clave pública `(n, e)` |
| `d` | Exponente privado → clave privada `(n, d)` |
| `m`, `c` | Mensaje en claro / cifrado |

Ejemplo propio: `p = 11`, `q = 7` → `n = 77`, `φ(n) = 10 × 6 = 60`. Trampa típica: confundir `n` con `φ(n)`.

Si alguien factoriza `n`, puede calcular `d` y romper la clave: por eso el tamaño importa. **Mínimo recomendado: 2048 bits.**

En CTF: RsaCtfTool, rsatool.

## 6. Diffie-Hellman

Dos partes acuerdan una clave secreta **sin enviarla nunca**.

**Analogía de la pintura:** color base público + un color secreto cada uno → cada uno manda su mezcla → cada uno agrega su secreto a la mezcla recibida → ambos obtienen el mismo color final; el espía, que solo vio las mezclas, no puede reproducirlo.

| Paso | Fórmula |
|---|---|
| Públicos | primo `p`, generador `g` |
| Privados | `a` (Alicia), `b` (Bob) |
| Claves públicas | `A = g^a mod p`, `B = g^b mod p` |
| Secreto compartido | Alicia: `B^a mod p` · Bob: `A^b mod p` (dan lo mismo) |

```bash
python3 -c "print(pow(3, 13, 29))"   # pow(base, exponente, módulo); la calculadora común redondea
```

- DH **acuerda** la clave; no cifra.
- No prueba identidad: vulnerable a *man-in-the-middle* si no se combina con firmas o certificados (RSA). TLS usa ambos.

## 7. SSH

- **El cliente verifica al servidor:** la huella (*fingerprint*) que se acepta con `yes`. Si cambia, posible *man-in-the-middle*.
- **El servidor verifica al cliente:** contraseña o clave.

```bash
ssh-keygen -t ed25519                 # crear par de claves
ssh-copy-id user@host                 # copiar solo la pública al servidor
ssh -i clave_privada user@host        # conectarse con una clave
ssh-keygen -l -f archivo              # ver huella y algoritmo de una clave
head -5 clave_privada                 # ver la cabecera
```

**Lo observado en el lab:**
- `Permissions 0644 ... are too open. This private key will be ignored.` → `chmod 600`. Con 644, cualquier usuario del sistema (por ejemplo una cuenta de servicio comprometida) puede leer la clave.
- `is not a key file` aun con 600: la clave estaba **protegida con contraseña**. La cabecera lo mostraba:

```text
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,<IV>
```

Una clave **asimétrica** protegida con cifrado **simétrico**. El resto del archivo es Base64 cifrado. Si la contraseña es débil, cae con `ssh2john` + John the Ripper + un diccionario.

**Lectura SOC — persistencia:** un atacante puede agregar su clave pública a `~/.ssh/authorized_keys` para volver sin contraseña (MITRE T1098.004). En un Linux comprometido se revisa el `authorized_keys` de cada usuario y de root, junto con crontabs y `.bashrc`. Auditoría: buscar claves privadas legibles por otros (`find / -name "id_*" -perm -044 2>/dev/null`).

## 8. Firmas digitales y certificados

**Firma digital:** se calcula el hash del documento y se cifra con la **privada** del firmante. Quien verifica descifra con la **pública** y compara con el hash del documento recibido. Si coinciden: autenticidad + integridad. Una imagen de firma pegada en un PDF no prueba nada.

**Certificados:** cadena de confianza.

```text
CA raíz (instalada en el navegador/sistema)
  └── CA intermedia
        └── certificado del sitio
```

Certificados TLS gratuitos: Let's Encrypt. Un candado HTTPS prueba que hablas con ese dominio, no que el dominio sea honesto.

## 9. PGP / GPG

Cifrado y firma de archivos y correos. Claves distintas para firmar (`[SC]`) y cifrar (`[E]`).

```bash
gpg --full-gen-key          # generar par
gpg --import clave.key      # importar al llavero
gpg --decrypt mensaje.gpg   # descifrar
```

**Lo observado:** el mensaje del lab estaba cifrado con una clave **RSA de 1024 bits**, bajo el mínimo recomendado (hallazgo de auditoría).

**Lectura SOC:** las firmas GPG verifican descargas (por ejemplo imágenes de Kali): si la firma no verifica con la clave oficial, el archivo pudo ser manipulado.

## 10. Ataques mencionados

- **Fuerza bruta:** probar todas las combinaciones.
- **Diccionario:** probar palabras probables de una lista (por ejemplo `rockyou.txt`).
- **Criptoanálisis:** estudio de cómo romper sistemas sin la clave.

## 11. Errores propios y autoevaluación

Cuestionario: 70 %. Los tres errores fueron del mismo tema:
- Elegir la clave equivocada para ocultar y para firmar → se adoptó la regla de dos pasos (sección 3).
- Confundir si 3DES es simétrico, en una pregunta negativa → truco de nombres ("ES" = simétrico; apellidos = asimétrico) y lectura de NO/EXCEPTO.

Otros: `cd..` sin espacio; opción y argumento siempre separados.

## 12. Preguntas de repaso

1. Diferencia entre autenticación y autenticidad.
2. Alicia quiere que solo Bob lea un archivo. ¿Con qué clave cifra?
3. Bob quiere probar que firmó un contrato. ¿Qué cifra y con qué clave?
4. Con `p = 13` y `q = 5`, calcula `n` y `φ(n)`.
5. ¿Qué pasa si alguien factoriza `n` en RSA?
6. ¿Para qué sirve Diffie-Hellman y cuál es su debilidad sola?
7. ¿Por qué SSH rechaza una clave privada con permisos 644?
8. ¿Qué indica `Proc-Type: 4,ENCRYPTED` en una clave?
9. ¿Por qué revisar `authorized_keys` en una investigación?
10. ¿Qué concluyes si ves una clave RSA de 1024 bits?

<details>
<summary><strong>Respuestas (abrir después de intentarlo)</strong></summary>

1. Autenticación: confirmar quién es la persona (login). Autenticidad: confirmar que un mensaje viene de esa persona (firma).
2. Con la pública de Bob.
3. El hash del contrato, con su propia clave privada.
4. `n = 65`; `φ(n) = 12 × 4 = 48`.
5. Puede calcular la clave privada `d` y descifrar o firmar como el dueño.
6. Acordar una clave secreta sin enviarla. Sola no autentica: vulnerable a man-in-the-middle.
7. Porque otros usuarios del sistema podrían leerla; exige 600 o más estricto.
8. Que la clave privada está protegida con contraseña (cifrada, por ejemplo con AES).
9. Porque una clave pública agregada por un atacante es una puerta trasera persistente sin contraseña.
10. Es un hallazgo: por debajo del mínimo recomendado de 2048 bits.

</details>
