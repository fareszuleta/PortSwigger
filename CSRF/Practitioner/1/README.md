# CSRF where token validation depends on request method — PortSwigger

![Plataforma](https://img.shields.io/badge/Plataforma-PortSwigger-orange)
![Dificultad](https://img.shields.io/badge/Dificultad-Pr%C3%A1ctica-yellow)
![Tipo](https://img.shields.io/badge/Tipo-CSRF-red)
![Estado](https://img.shields.io/badge/Estado-Completada-success)

Writeup del laboratorio **CSRF where token validation depends on request method** de PortSwigger Web Security Academy.

## Técnicas utilizadas

- Interceptación y análisis de peticiones HTTP con Burp Suite
- Manipulación del método HTTP (POST → GET) para evadir validación de token
- Crafting de payload HTML autoejecutable
- Despliegue de exploit en exploit server

## Flujo resumido

```
Login (wiener:peter) → Interceptar POST con token CSRF → Cambiar método a GET
→ Confirmar bypass de validación → Crear payload HTML → Exploit server
→ Entregar a víctima → Email de víctima cambiado
```

## Archivos

| Archivo | Descripción |
|---------|-------------|
| `CSRF_token_validation_github.md` | Writeup completo |

## Referencias

- [PortSwigger — CSRF where token validation depends on request method](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-validation-depends-on-request-method)
- [OWASP — CSRF](https://owasp.org/www-community/attacks/csrf)
- [PortSwigger — CSRF](https://portswigger.net/web-security/csrf)
