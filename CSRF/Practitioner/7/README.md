# PortSwigger Lab: SameSite Strict bypass via client-side redirect

![PortSwigger](https://img.shields.io/badge/Plataforma-PortSwigger%20Academy-blue)
![Dificultad](https://img.shields.io/badge/Dificultad-Dif%C3%ADcil-ff4444)
![Tipo](https://img.shields.io/badge/Tipo-CSRF%20SameSite%20Strict%20Bypass-red)
![Estado](https://img.shields.io/badge/Estado-Completada-success)

Writeup del laboratorio de PortSwigger sobre CSRF bypasseando SameSite Strict con client-side redirect + path traversal.

## Técnicas utilizadas

- CSRF (Cross-Site Request Forgery)
- SameSite Strict bypass
- Client-side redirect exploitation
- Path traversal (./)
- IDOR (Insecure Direct Object Reference)
- URL parameter encoding (%26)
- GET en POST endpoint

## Flujo resumido

```
SameSite=Strict bloquea CSRF cross-site
→ Buscar redirect mismo-dominio controlable
→ Path traversal al endpoint de cambio
→ URL encode parámetros anidados
→ GET request bypasea SameSite Strict (same-origin)
→ Payload: document.location con URL armada
```

## Vulnerabilidad clave

**SameSite Strict** protege contra CSRF cross-site.

**PERO:** Los redirects **desde el mismo dominio** son tratados como same-origin.

Si hay un redirect controlable:
1. Atacante usa redirect (GET) = same-origin
2. Cookie se envía (SameSite Strict permite)
3. Servidor genera segundo redirect
4. Cookie persiste en follow-up

## Técnicas combinadas

- **Redirect no sanitizado** en `/post/comment/confirmation?postId=X`
- **Path traversal** `postId=../my-account/change-email`
- **GET aceptado** en POST endpoint
- **URL encoding** `%26` para & en parámetros anidados
- **IDOR** cualquier postId accesible
- **Sin CSRF token** en el endpoint

## Payload

```html
<script>
  document.location = "https://target.com/post/comment/confirmation?postId=../my-account/change-email?email=attacker%40attacker.com%26submit=1"
</script>
```

## Timing del bypass

1. Atacante envía GET (mismo dominio) ✅
2. SameSite Strict: "Es same-origin, envío cookie"
3. Servidor recibe, ve path traversal ../my-account/change-email
4. Genera redirect a /my-account/change-email?email=...
5. Navegador sigue redirect (aún same-origin)
6. Cookie persiste
7. Email cambiado

## Por qué es peligroso

- SameSite Strict parece suficiente
- Pero redirects mal configurados bypasean
- Combinación de fallos menores = crítico
- Sin tokens CSRF, toda acción es vulnerable

## Referencias

- [PortSwigger — CSRF](https://portswigger.net/web-security/csrf)
- [MDN — SameSite Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [OWASP — Open Redirect](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)
- [Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
