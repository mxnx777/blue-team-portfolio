# TryHackMe — Cryptography Basics

**Fecha:** 25 de noviembre de 2026 · **Ruta:** Cyber Security 101 → Cryptography Basics
**Propósito:** vocabulario y fundamentos de criptografía, cifrado simétrico vs asimétrico, XOR y módulo, con lectura SOC.

> Sin flags ni respuestas literales de la sala. Los ejemplos numéricos y de texto son propios.

## 1. Diccionario de siglas

| Sigla | Nombre en inglés | Significado |
|---|---|---|
| PCI DSS | Payment Card Industry Data Security Standard | Norma de seguridad para datos de tarjetas de pago |
| HIPAA | Health Insurance Portability and Accountability Act | Ley de EE. UU. sobre datos de salud |
| GDPR / RGPD | General Data Protection Regulation | Reglamento europeo de protección de datos |
| AES | Advanced Encryption Standard | Estándar de cifrado simétrico actual |
| DES / 3DES | Data Encryption Standard / Triple DES | Cifrados simétricos obsoletos |
| RSA | Rivest–Shamir–Adleman | Algoritmo asimétrico (apellidos de sus autores) |
| ECC | Elliptic Curve Cryptography | Criptografía de curva elíptica |
| XOR | Exclusive OR | O exclusivo |

## 2. Para qué sirve la criptografía

Protege **confidencialidad** (nadie más lee), **integridad** (nadie modifica) y **autenticidad** (viene de quien dice).

- **Datos en reposo** (*at rest*): guardados (base de datos, disco).
- **Datos en tránsito** (*in transit*): viajando por la red.
- Las normas exigen proteger ambos. Ejemplos: tarjetas → PCI DSS; salud en EE. UU. → HIPAA; UE → GDPR. En Chile: Ley 21.719 (protección de datos personales; los datos de salud son sensibles) y Ley 21.663 (Ley Marco de Ciberseguridad).

## 3. Vocabulario

| Inglés | Español | Qué es |
|---|---|---|
| Plaintext | Texto plano | Dato legible antes de cifrar |
| Ciphertext | Texto cifrado | Dato ilegible después de cifrar |
| Cipher | Algoritmo de cifrado | El método |
| Key | Clave | El secreto que usa el método |
| Encryption | Cifrado | Plaintext → ciphertext |
| Decryption | Descifrado | Ciphertext → plaintext |

**Principio de Kerckhoffs:** el algoritmo puede ser público; **lo único secreto es la clave**. AES es público y sigue siendo seguro.

## 4. Cifrados históricos: César y fuerza bruta

César desplaza cada letra un número fijo de posiciones. Ejemplo propio con clave 3: `HOLA` → `KROD` (alfabeto inglés de 26 letras).

Solo hay **25 claves útiles**, así que se prueban todas (**fuerza bruta**):

```bash
c="TEXTOCIFRADO"; A=ABCDEFGHIJKLMNOPQRSTUVWXYZ
for k in $(seq 1 25); do echo "$k $(echo $c | tr A-Z "${A:$((26-k))}${A:0:$((26-k))}")"; done
```

- `tr A-Z "..."` reemplaza cada letra por la que está `k` posiciones antes.
- Se lee la única línea con sentido. La clave no tiene por qué ser 3.

Otros históricos: Vigenère (s. XVI), Enigma (II Guerra Mundial), one-time pad.

## 5. Simétrico vs asimétrico

| | Simétrico | Asimétrico |
|---|---|---|
| Claves | Una, compartida | Par: pública + privada |
| Velocidad | Rápido | Lento |
| Problema | Cómo hacer llegar la clave al otro | Lento para muchos datos |
| Ejemplos | AES; DES y 3DES obsoletos | RSA, ECC; Diffie-Hellman (intercambio de claves) |

**Truco de nombres:** si termina en **"ES"** (*Encryption Standard*: DES, 3DES, AES) → simétrico. Si tiene **apellidos de personas** (RSA, Diffie-Hellman) → asimétrico.

**Analogía:** simétrico = caja fuerte con una llave que hay que copiarle al otro. Asimétrico = candados abiertos que regalas (pública) y una sola llave que guardas (privada).

**En la práctica se usan juntos (cifrado híbrido):** el asimétrico acuerda una clave de forma segura y el simétrico cifra los datos. Es lo que hace TLS en cada conexión HTTPS.

- DES (1977, clave de 56 bits) se rompió en menos de 24 h en 1999.
- 3DES es DES tres veces; obsoleto desde 2019.
- AES (2001), claves de 128, 192 o 256 bits.

## 6. Matemática básica

**XOR (⊕):** 1 si los bits son distintos, 0 si son iguales.

```text
1100 ⊕ 0110 = 1010
```

Propiedad clave: `P ⊕ K = C` y `C ⊕ K = P`. Aplicar la misma clave dos veces devuelve el original; por eso XOR está dentro de los cifrados simétricos.

**Módulo (%):** el resto de una división, como un reloj. `17 % 5 = 2`, `24 % 12 = 0`. No es reversible: si `x % 5 = 4`, `x` puede ser 4, 9, 14... Base de las operaciones "fáciles de ida, imposibles de vuelta".

```bash
python3 -c "print(bin(0b1100 ^ 0b0110), 17 % 5)"   # bin() omite ceros a la izquierda
```

Sin Python: calculadora de Windows en modo Programador.

## 7. Lectura SOC

- **DES o 3DES en una configuración** (por ejemplo TLS de un sistema antiguo) es un hallazgo: obsoletos.
- **Malware y XOR:** muchos ofuscan textos (IP, URL, comandos) con XOR de un byte para que `strings` no los muestre; se rompe con fuerza bruta, como el César.
- **Pocas claves posibles = inseguro.** César: 25. AES-256: 2²⁵⁶. Es el mismo principio de la fuerza bruta contra un login SSH.

## 8. Errores propios

| Error | Corrección |
|---|---|
| Responder sin leer la palabra clave de la pregunta | Escribir primero "Clave:" y luego la respuesta |
| Confundir "&" y "!= 0" con "distinto de" en otro contexto | Leer qué compara realmente la expresión |

## 9. Preguntas de repaso

1. ¿Qué debe ser secreto según Kerckhoffs?
2. ¿Qué diferencia hay entre datos en reposo y en tránsito?
3. ¿Por qué César es inseguro?
4. Nombra un algoritmo simétrico actual y dos obsoletos.
5. ¿Qué problema tiene el cifrado simétrico que resuelve el asimétrico?
6. ¿Qué es el cifrado híbrido y dónde se usa?
7. Calcula `1010 ⊕ 0011`.
8. Calcula `29 % 6`.
9. ¿Por qué el módulo no es reversible?
10. ¿Qué haría un analista si ve un texto ofuscado con XOR?

<details>
<summary><strong>Respuestas (abrir después de intentarlo)</strong></summary>

1. Solo la clave. El algoritmo puede ser público (AES lo es).
2. Reposo: guardados (disco, base de datos). Tránsito: viajando por la red.
3. Tiene solo 25 claves; se prueban todas en segundos.
4. AES; DES y 3DES.
5. Cómo hacerle llegar la clave al otro sin que la intercepten.
6. Asimétrico para acordar la clave + simétrico para los datos. TLS/HTTPS.
7. `1001`.
8. `5` (29 = 4 × 6 + 5).
9. Muchos números dan el mismo resto: `x % 5 = 4` tiene infinitas soluciones.
10. Probar XOR con las 256 claves de un byte (fuerza bruta) y buscar texto legible.

</details>
