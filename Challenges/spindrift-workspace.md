# Spindrift Workspace — The Self-Describing Session That Promoted Bob

**Category:** COOKIES  
**Difficulty:** EASY  
**Solves:** 34  
**Flag:** `WEBVERSE{74ef0cde...906d}`

---

## The Setup

Spindrift Workspace is a small project-management tool for freelance creatives — two founders, no venture money, no AI features. They built their own session layer one weekend "because we don't need a whole library for this." A junior contractor noted in Linear that the cookie was "stateless and self-describing" and shipped it.

The login page at `/login` has demo credentials displayed right on the form: a demo seat for `bob@example` with password `bob`.

---

## Recon

Logging in with `bob@example` / `bob` redirects to `/dashboard` — a standard Kanban workspace. But the category is COOKIES and the challenge slug is `honor-system`, which hints at an honor-system trust model for authorization.

Checking the browser cookies reveals the session token:

| Cookie | Value |
|--------|-------|
| `sw_session` | `eyJ1aWQiOjQyLCJlbWFpbCI6ImJvYkBleGFtcGxlIiwicm9sZSI6Im1lbWJlciIsImV4cCI6MTc4Mjg1NjM4NH0=` |

The value is base64-encoded JSON. Decoding it:

```json
{"uid":42,"email":"bob@example","role":"member","exp":1782856384}
```

The `role` field is embedded in a stateless, unsigned cookie — entirely client-trust-based authorization.

---

## Exploitation

### Step 1: Forge an Admin Session Cookie

Change `role` from `member` to `admin` and re-encode:

```json
{"uid":42,"email":"bob@example","role":"admin","exp":9999999999}
```

Base64: `eyJ1aWQiOjQyLCJlbWFpbCI6ImJvYkBleGFtcGxlIiwicm9sZSI6ImFkbWluIiwiZXhwIjo5OTk5OTk5OTk5fQ==`

Set the forged cookie from the browser console:

```javascript
document.cookie = 'sw_session=eyJ1aWQiOjQyLCJlbWFpbCI6ImJvYkBleGFtcGxlIiwicm9sZSI6ImFkbWluIiwiZXhwIjo5OTk5OTk5OTk5fQ==; path=/';
```

### Step 2: Access the Admin Panel

After setting the cookie, an **Admin** link appears in the navigation bar pointing to `/admin/billing`.

Navigating there loads the billing & reconciliation dashboard — an operator view of monthly revenue, active workspaces, churn, and the day's invoice ledger.

### Step 3: Read the Flag

The admin billing page displays the reconciliation reference:

> **RECONCILIATION REFERENCE**
>
> Daily reference token, matched against the Stripe reconciliation report at 23:55 UTC.
>
> **TODAY'S REFERENCE**
>
> `WEBVERSE{74ef0cde9d85ff9e555cd6b8d33e906d}`

No bypass, no injection, no brute-force — just decode, tweak, re-encode.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **Stateless session cookies are insecure without signing** | A base64-encoded JSON cookie is not a session — it's a suggestion. Anyone can decode, modify, and re-encode it. |
| **Client-side roles are not authorization** | If `role: member` can become `role: admin` by editing a cookie, there is no authorization. |
| **Demo credentials are a starting point, not an endpoint** | The demo seat gave us a foothold. The real vulnerability was in the session design. |
| **The challenge name tells the story** | "Honor System" — the server trusts the client to be honest about its own role. |

**Remediation checklist:**
- [ ] Use server-side sessions with opaque, random session IDs (not self-describing tokens)
- [ ] If JWT-like tokens are required, sign them with a strong secret and validate the signature on every request
- [ ] Never store role/authorization data in client-modifiable storage (cookies, localStorage)
- [ ] Enforce authorization server-side on every protected endpoint, not just in the frontend routing

---

*"Two founders, eighteen months in, no marketing budget, no AI features. They built their own session layer one weekend."*

**Flag:** `WEBVERSE{74ef0cde...906d}`
