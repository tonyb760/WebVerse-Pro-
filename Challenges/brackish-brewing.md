# Brackish Brewing Co. — The X-Forwarded-For That Opened the Staff Room

**Category:** BASICS  
**Difficulty:** EASY  
**Solves:** 60+  
**Flag:** `WEBVERSE{2653d230...27ca3c}`

---

## The Setup

Brackish Brewing Co. is a small craft brewery. Their site has a public-facing storefront and a staff portal at `/staff` for shift scheduling and inventory.

The `admin` account is disabled — there's no password to crack, no SQL injection, no hidden endpoint. The only thing standing between you and the staff portal is an IP check.

---

## Recon

Navigating to `/staff` returns a page that reads:

> **Staff Portal — Restricted Access**
>
> This portal is only accessible from the brewery's internal network. If you are a staff member, please connect to the brewery Wi-Fi and try again.

No login form, no token field — just a gate that checks the originating IP address. The server is checking the remote address of the HTTP request and rejecting anything outside a known internal range (likely `127.0.0.0/8` or `10.0.0.0/8`).

---

## Exploitation

### Step 1: Add the Spoofed Header

Since the server checks the connection IP but doesn't cryptographically validate it, we can inject an `X-Forwarded-For` header to trick the server into thinking the request came from `127.0.0.1`.

Using browser dev tools, intercept the request or use `fetch()` with a custom header:

```javascript
fetch('/staff', {
  headers: { 'X-Forwarded-For': '127.0.0.1' }
}).then(r => r.text()).then(console.log);
```

Or simply navigate to the staff page after setting up a browser extension/interception proxy that adds the header.

### Step 2: Access the Staff Portal

The server accepts the spoofed header and returns the staff dashboard:

```html
<h1>Staff Dashboard</h1>
<p>Welcome to the Brackish Brewing staff portal.</p>
```

The response includes an internal notice with the flag.

### Step 3: Read the Flag

The staff portal page displays:

> **Internal Notice — Q2 Inventory Reconciliation**
>
> All staff: please confirm your shift preferences for the June bottling run before the 15th.
>
> **Reference token: `WEBVERSE{2653d2302991ae65db9864cb4527ca3c}`**

The flag was sitting behind a single header. No authentication, no session, no rate limit — just `X-Forwarded-For: 127.0.0.1`.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **IP-based access control is trivially bypassed** | `X-Forwarded-For`, `X-Real-IP`, `X-Client-IP`, and `Forwarded` are all client-controlled headers. Relying on any of them for access control is equivalent to no access control. |
| **Check the `/staff` or `/admin` path first** | Internal portals are often hidden but not authenticated — just gated behind a simple check that can be bypassed. |
| **Always try `127.0.0.1`** | The most common internal IP range used in these checks. If that fails, try `10.0.0.1`, `192.168.1.1`, or other private ranges. |
| **No auth needed** | If the only gate is an IP check and you can control the IP header, you're already in. |

**Remediation checklist:**
- [ ] Never use client-controlled headers (`X-Forwarded-For`, etc.) for access control
- [ ] Use a proper VPN or network-level authentication for internal portals
- [ ] Strip or re-verify `X-Forwarded-For` at the reverse proxy before passing it to the application
- [ ] If an internal portal must exist on a public server, require real authentication (session + credentials)

---

*"Brackish Brewing's staff portal is restricted to internal IPs. The restriction is an HTTP header."*

**Flag:** `WEBVERSE{2653d230...27ca3c}`
