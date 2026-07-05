# PortSwigger Lab: Manipulating WebSocket messages to exploit vulnerabilities

![PortSwigger](https://img.shields.io/badge/Platform-PortSwigger%20Academy-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-yellow)
![Type](https://img.shields.io/badge/Type-WebSocket%20XSS-red)
![Status](https://img.shields.io/badge/Status-Completed-success)

Writeup of the PortSwigger lab on XSS through WebSocket message manipulation.

## Techniques Used

- WebSocket interception (Burp Suite)
- WebSocket message manipulation
- XSS via WebSocket
- Real-time communication exploitation
- JSON payload modification

## Attack Summary

```
Live chat WebSocket → Send normal message
→ Intercept in Burp (WebSocket frames)
→ Change JSON payload to XSS
→ Forward manipulated message
→ Server broadcasts to agent
→ alert() executed in support agent's browser
```

## WebSocket vs HTTP

| Aspect | HTTP | WebSocket |
|--------|------|-----------|
| Protocol | Request-Response | Bidirectional persistent |
| Intercept | Easy (default) | Requires activation |
| Real-time | No | Yes |
| Message | URL/body | JSON |

## Vulnerability

The server **does not validate or sanitize** WebSocket message content. Messages are rendered directly in the agent's browser without escaping HTML.

## XSS Payload

```json
{
  "message": "<img src=1 onerror='alert(1)'>"
}
```

Executes `alert(1)` in support agent's browser.

## Steps

1. Access Live chat
2. Send normal message ("hi")
3. Intercept WebSocket in Burp (activate intercept)
4. Modify JSON with XSS payload
5. Forward the message
6. alert() executed in agent's browser

## Why It Works

- No content validation
- No HTML sanitization
- No Content Security Policy
- Implicit trust in client messages
- Real-time broadcasting

## Alternative Payloads

```html
<img src=1 onerror='alert(1)'>
<script>alert(1)</script>
<svg onload='alert(1)'>
<body onload='alert(1)'>
```

All work without validation.

## References

- [PortSwigger — WebSocket](https://portswigger.net/web-security/websockets)
- [OWASP — XSS Prevention](https://owasp.org/www-community/attacks/xss/)
- [MDN — WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
