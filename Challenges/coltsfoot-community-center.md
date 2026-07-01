# Coltsfoot Community Center — The Staff Side Wasn't So Hidden

**Category:** IDOR  
**Difficulty:** EASY  
**Solves:** 53  
**Flag:** `WEBVERSE{6a57fd0b...247902}`

---

## The Setup

Coltsfoot Community Center is run by a three-member board of retirees. They took over the operations site from a volunteer web-developer in 2019. The board insists the "staff side" of the site has always been hidden — no one outside the building has ever needed it.

Except the developer never actually protected it. They just didn't link to it from the navigation.

---

## Recon

The public site has the usual pages: home, classes, board, volunteer, donate, sign in. Nothing unusual on the surface. No hidden HTML comments, no secrets in headers.

But the description hints at a "staff side" — and checking common hidden paths reveals it immediately.

---

## Exploitation

### Step 1: Discover Hidden Endpoints

A quick scan of common hidden paths:

```
/staff            → 200 ✅
/staff/dashboard  → 200 ✅
/admin            → 404
/backstage        → 404
```

The `/staff` endpoint redirects to `/staff/dashboard`.

### Step 2: Read the Dashboard

The staff dashboard shows the daily reconciliation screen for the community center, including:

```
RECONCILIATION REFERENCE
WEBVERSE{6a57fd0b...247902}
Paste into the deposit slip when closing out the drawer.
```

The flag is sitting in plain text on an unauthenticated staff dashboard. No login, no IDOR parameter manipulation — just a hidden path that was never locked down.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **"Hidden" is not "secure"** | Not linking to a page in the nav doesn't mean it's protected. Any path scanner will find it. |
| **Always check `/staff`, `/admin`, `/dashboard`** | These are the most common hidden paths in web apps. |
| **Authenticate all admin endpoints** | If a page shows reconciliation data, it should require a login — regardless of whether the nav links to it. |

**Remediation checklist:**
- [ ] Audit all routes for authentication middleware
- [ ] Don't rely on "security through obscurity" for admin pages
- [ ] Use proper role-based access control for all staff endpoints

---

*"The staff side was hidden. Just not very well."*

**Flag:** `WEBVERSE{6a57fd0b...247902}`
