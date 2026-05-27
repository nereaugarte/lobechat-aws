# TLS Validation Checklist

Public hostname: **https://108.130.141.156.sslip.io**  
Student: nerea.ugarte@alumni.esade.edu  
Date validated: 2026-05-27  
Certificate issuer: ZeroSSL (via Caddy / Let's Encrypt ACME)

| Check | Status | Notes |
|---|---|---|
| Casdoor login flow completes from public URL | ✅ PASS | Casdoor SSO login via `https://108.130.141.156.sslip.io` completes with no cookie/redirect_uri errors |
| LobeChat chat streaming works | ✅ PASS | Response tokens arrive incrementally (SSE confirmed via `openrouter/free` model) |
| At least one MCP tool invoked and returns a result | ✅ PASS | `mcphub-filesystem` → `filesystem-list_directory` invoked from chat, returned `/tmp` directory listing |
| Direct connection to EC2 origin on port 47000 is rejected | ✅ PASS | `curl --max-time 5 http://108.130.141.156:47000/` times out — port blocked by Security Group |
| Browser shows valid certificate chain | ✅ PASS | No browser warning; issuer is ZeroSSL (public CA via Caddy ACME); HTTP/2 confirmed |

## Evidence

### curl -sI https://108.130.141.156.sslip.io/
```
HTTP/2 307
alt-svc: h3=":443"; ma=2592000
date: Wed, 27 May 2026 11:37:26 GMT
location: /chat
via: 1.1 Caddy
```

### Port 47000 blocked
```
$ curl -v --max-time 5 http://108.130.141.156:47000/
*   Trying 108.130.141.156:47000...
* Connection timed out after 5005 milliseconds
curl: (28) Connection timed out after 5005 milliseconds
```

### MCP tool call
See `chat-mcp.png` — `mcphub-filesystem` called `filesystem-list_directory(/tmp)` and returned file listing inline in chat.

## Screenshots
- `lobechat-https.png` — LobeChat home page, logged in, HTTPS padlock visible, ESADE email visible
- `chat-mcp.png` — chat reply with MCP tool call result rendered
