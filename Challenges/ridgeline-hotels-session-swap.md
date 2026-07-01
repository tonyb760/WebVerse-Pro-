# Ridgeline Hotels — The Session Swap That Opened the Admin Console

**Category:** BASICS  
**Difficulty:** EASY  
**Solves:** 60+  
**Flag:** `WEBVERSE{006ff7a3...d807d7b}`

---

## The Setup

Ridgeline Hotels runs a crew portal at `/` where staff sign in to view schedules, payroll, and property information. There's also an admin console at `/admin` for property managers.

The challenge is called "Session Swap" — the hint is in the name: roles and permissions are managed client-side, and all we need to do is swap our session from `user` to `admin`.

---

## Recon

The login page accepts common credentials — `admin`/`admin` works out of the box, giving you access to the crew dashboard. But clicking "Admin Console" in the sidebar returns:

> **Access Denied**

Checking the browser cookies reveals why:

| Cookie | Value | Domain |
|--------|-------|--------|
| `role` | `user` | challenges.webverselabs-pro.com |
| `badge` | `RD-04412` | challenges.webverselabs-pro.com |

The `role` cookie controls authorization, and it's stored entirely on the client side with no HttpOnly flag, no signing, and no server-side validation.

---

## Exploitation

### Step 1: Swap the Role Cookie

Using the browser developer console, change the `role` cookie from `user` to `admin`:

```javascript
document.cookie = 'role=admin; path=/';
```

### Step 2: Access the Admin Console

Navigating to `/admin` now loads the admin dashboard instead of the access-denied page:

```
GET /admin → 200 OK ✅
```

### Step 3: Read the Flag

The admin console displays an internal memo:

> **INTERNAL MEMO · FOR ADMINISTRATIVE REVIEW ONLY**
>
> **Q2 Internal Reference Token**
>
> Quarterly internal reference token issued to property administrators for
> cross-system reconciliation. Do not share outside the Ridgeline admin tier.
>
> **Current token: `WEBVERSE{006ff7a3d21b3222702478d5fd807d7b}`**

The flag was sitting behind a single cookie value — `user` → `admin`.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **Client-side roles are not security** | If authorization is enforced via a cookie or localStorage value that the client can modify, it's not authorization at all — it's a suggestion. |
| **Always check the HttpOnly flag** | If a sensitive cookie (like `role`) is not HttpOnly, it can be modified from JavaScript. This is a design flaw. |
| **The challenge name tells the story** | "Session Swap" = swap the `role` from `user` to `admin`. |
| **`admin`/`admin` is worth trying** | Default credentials are a common finding — always try obvious combinations before looking deeper. |

**Remediation checklist:**
- [ ] Never enforce role-based access control via client-side cookies
- [ ] Use server-side sessions or signed/encrypted tokens for authorization
- [ ] Set HttpOnly and Secure flags on all session cookies
- [ ] Validate authorization on every protected endpoint, not just in the frontend

---

*"Ridgeline's admin console is restricted. The restriction is a cookie."*

**Flag:** `WEBVERSE{006ff7a3...d807d7b}`
