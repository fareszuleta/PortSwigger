# PortSwigger Lab: Manipulating the WebSocket handshake to exploit vulnerabilities

![PortSwigger](https://img.shields.io/badge/Platform-PortSwigger%20Academy-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Type](https://img.shields.io/badge/Type-WebSocket%20Filter%20Bypass-red)
![Status](https://img.shields.io/badge/Status-Completed-success)

Writeup of the PortSwigger lab on bypassing aggressive XSS filters in WebSocket communications.

## Techniques Used

- WebSocket message manipulation
- XSS filter bypass (multiple layers)
- IP blacklist evasion (X-Forwarded-For header)
- Case variation bypass
- Template literal injection (backticks)
- Burp Suite Repeater
- Burp Suite Match and Replace

## Attack Summary

```
Live chat WebSocket → Send XSS payload
→ Filter blocks, IP blacklisted
→ Spoof IP with X-Forwarded-For header
→ Reconnect, send case-varied payload
→ Use backticks instead of parentheses
→ Filter bypassed, alert() executed
```

## Key Vulnerabilities

| Vulnerability | Impact |
|---------------|--------|
| **Aggressive but flawed filter** | Blocks common patterns but not case variation + backticks |
| **IP-based blocking** | Can be bypassed with X-Forwarded-For header |
| **No whitelist validation** | Server trusts X-Forwarded-For without verification |
| **Template literal oversight** | Backtick syntax not blocked by basic filters |

## Filter Bypass Layers

### Layer 1: IP Blacklist
- **Protection:** Server blocks IP after detecting attack
- **Bypass:** X-Forwarded-For header (spoofs IP address)

### Layer 2: Event Handler Filter
- **Protection:** Filter blocks `onerror`, `onclick`, etc.
- **Bypass:** Case variation (`OnErrOr`, `ONERROR`)

### Layer 3: Function Call Filter
- **Protection:** Filter blocks parentheses in `alert(1)`
- **Bypass:** Template literals `` alert`1` `` (valid JavaScript)

## Complete Payload

```json
{
  "message": "<img src=1 OnErrOr='alert`1`'>"
}
```

**Bypass breakdown:**
- `OnErrOr` bypasses case-sensitive event handler filter
- `alert`1`` (backticks) bypasses function call filter
- X-Forwarded-For header bypasses IP blacklist

## X-Forwarded-For Header

```
Original client: 192.168.1.100
Server sees: X-Forwarded-For: 10.0.0.1
Result: Server thinks 10.0.0.1 made the request
```

Used by proxies but can be spoofed.

## Template Literals

```javascript
alert(1)      // Standard function call - blocked by filter
alert`1`      // Template literal - bypasses filter
alert`${1}`   // Can include expressions
```

Both execute the same code but different syntax.

## JavaScript Template Literals

Backticks allow string interpolation and tag functions in JavaScript.

Filters that look for `alert(` don't catch `alert`` - syntactically different.

## Why It Works

- **Trust in proxy headers:** X-Forwarded-For assumed legitimate
- **Case-sensitive blacklists:** Filter doesn't account for case variation
- **Syntax overlooked:** Template literals are valid but uncommon
- **No rate limiting:** Can reconnect infinitely with different IPs
- **WebSocket permissive:** Less strict validation than HTTP

## Burp Suite Tools Used

1. **WebSocket tab:** Capture and modify WebSocket messages
2. **Repeater:** Send modified messages and view responses
3. **Match and Replace:** Automatically add X-Forwarded-For to all requests

## References

- [PortSwigger — WebSocket](https://portswigger.net/web-security/websockets)
- [MDN — X-Forwarded-For Header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For)
- [JavaScript Template Literals](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals)
- [XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
