# TLS Validation Checklist

Public hostname: **https://54.217.113.13.sslip.io**  
Student: nerea.ugarte@alumni.esade.edu  
Date validated: 2026-05-31  
Certificate issuer: ZeroSSL (via Caddy / Let's Encrypt ACME)

| Check | Status | Notes |
|---|---|---|
| Casdoor login flow completes from public URL | ✅ PASS | Casdoor SSO login via `https://54.217.113.13.sslip.io` completes with no cookie/redirect_uri errors — see `login.png` |
| LobeChat chat streaming works | ✅ PASS | Response tokens arrive incrementally (SSE confirmed via `openrouter/free` model) — see `chat.png` |
| At least one MCP tool invoked and returns a result | ✅ PASS | `mcphub-filesystem` → `filesystem-list_directory(/tmp)` invoked from chat, returned directory listing inline — see `mcp-call.png` |
| File upload to MinIO from chat works | ✅ PASS | PDF (4.5 MB) uploaded and processed via MinIO S3-compatible storage — see `file-upload.png` |
| Direct connection to EC2 origin on port 47000 is rejected | ✅ PASS | `curl --max-time 5 http://54.217.113.13:47000/` times out — port blocked by AWS Security Group |
| Browser shows valid certificate chain | ✅ PASS | No browser warning; issuer is ZeroSSL (public CA via Caddy ACME); HTTP/2 confirmed in curl output |

## Evidence

### curl -sI https://54.217.113.13.sslip.io/
```
$ curl -sI https://54.217.113.13.sslip.io/
HTTP/2 307
alt-svc: h3=":443"; ma=2592000
date: Sun, 31 May 2026 13:50:35 GMT
location: /chat
via: 1.1 Caddy
```

HTTP/2 response, `via: 1.1 Caddy` confirms the reverse proxy is serving HTTPS. `307` redirects to `/chat` — expected LobeChat behaviour.

### Port 47000 blocked
```
$ curl -v --max-time 5 http://54.217.113.13:47000/
*   Trying 54.217.113.13:47000...
* Connection timed out after 5002 milliseconds
curl: (28) Connection timed out after 5002 milliseconds
```

Direct origin access is impossible — the AWS Security Group allows inbound only on ports 22, 80, and 443.

### Casdoor login
See `login.png` — browser at `https://54.217.113.13.sslip.io` shows Casdoor login form with no certificate warning. ESADE email confirmed in terminal alongside.

### Chat streaming
See `chat.png` — user `nerea.ugarte@alumni.esade.edu` logged in, chat response streamed incrementally via `openrouter/free`. MCP plugin badge (`mcphub-filesystem`) visible in chat toolbar.

### File upload to MinIO
See `file-upload.png` — `article_b.pdf` (4.5 MB) attached in chat and summarised by the model. Confirms `S3_ENDPOINT=http://minio:9000` path works: file stored in MinIO bucket `lobe`, retrieved by LobeChat for vision/document processing.

### MCP tool call
See `mcp-call.png` — `mcphub-filesystem` plugin called `filesystem-list_directory` with `path=/tmp`. Tool returned:
```
- fastembed_cache (directory)
- lobechat-mcp-evidence.txt (file)
- node-compile-cache (directory)
- uv-setuptools-c62782e1bddd45af.lock (file)
```
Result rendered inline in chat. Model: `openai/gpt-oss-120b:free` via OpenRouter. Confirms full MCP transport chain: LobeChat → MCPHub → filesystem MCP server → response back to chat.

## Screenshots
- `login.png` — Casdoor SSO login page at public HTTPS URL, no certificate warning
- `chat.png` — LobeChat chat, logged in as ESADE user, streaming response visible, MCP plugin active
- `file-upload.png` — PDF file uploaded from chat, processed via MinIO storage
- `mcp-call.png` — MCP tool `filesystem-list_directory` invoked and result returned inline in chat
