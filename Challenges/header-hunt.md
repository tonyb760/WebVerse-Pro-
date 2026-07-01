# Header Hunt — What the Debug Branch Left Behind

**Category:** BASICS  
**Difficulty:** EASY  
**Solves:** 92  
**Flag:** `WEBVERSE{575d5d00...e26a46f}`

---

## The Setup

Arc Logistics, a mid-sized regional freight carrier, launched a public shipment tracking site. The build team rushed to hit the Q2 deadline and merged a debugging branch the night before launch. Nobody asked what got left behind.

The phrase *"Header Hunt"* tells you exactly where to look: **HTTP headers**.

---

## Recon

The lab is a clean tracking portal for Arc Logistics — search by tracking number, check shipment status. Nothing unusual in the page content, no hidden comments, no cookies with secrets.

But the response headers tell a different story.

---

## Exploitation

### Step 1: Inspect Response Headers

Using the browser's fetch API to check headers:

```javascript
fetch('/').then(r => {
  r.headers.forEach((v, k) => console.log(k, v));
});
```

Or check directly in the browser DevTools Network tab.

One header stands out from the rest:

```
x-internal-order-ref: WEBVERSE{575d5d00...e26a46f}
```

The debugging branch left an internal order reference header that was never removed before launch — and it contains the flag.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **HTTP response headers are a common hiding spot** | Custom headers like `X-Debug-*`, `X-Internal-*`, or `X-Dev-*` are often left behind from development builds |
| **Always check headers first in "Header" challenges** | The challenge name is a spoiler — if it mentions headers, the flag is almost certainly in a header |
| **A quick `curl -I` or `fetch()` can save minutes** | Don't bother deep-diving the page HTML — check the metadata first |

**Remediation checklist:**
- [ ] Audit custom HTTP headers before production deploys
- [ ] Remove debug/internal headers from production builds
- [ ] Use a `Content-Security-Policy` to restrict what leaks

---

*"Arc Logistics ships freight. The flag shipped in a header."*

**Flag:** `WEBVERSE{575d5d00...e26a46f}`
