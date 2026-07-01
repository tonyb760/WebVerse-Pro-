# Lake Forks Permits — The Consultant's Default Credentials

**Category:** AUTH  
**Difficulty:** EASY  
**Solves:** 47+  
**Flag:** `WEBVERSE{2746fa85...5818}`

---

## The Setup

Lake Forks County runs a permits and licensing portal where staff can access the records system, view daily totals, and reconcile the permit log. The login page at `/login` is the only thing standing between the public and the staff records.

The staff portal was redesigned in 2018 by a consultant (Wescroft Civic Consulting) who set the login credentials as a temporary default and noted *"RESET BEFORE GOING LIVE"* in the project binder. The binder was never opened.

---

## Recon

The login page has a standard form with username (placeholder: `e.g. ebjornson`) and password fields. The footer contains a telling note:

> **For municipal staff:** log in via the records portal.  
> *(Default password reset Mar 2017 — see binder.)*

The challenge description confirms the vulnerability: the credentials were set as a temporary default during the 2018 redesign and never changed.

---

## Exploitation

### Step 1: Try Default Credentials

The consultant never rotated the default `admin` account:

```
POST /login
username=admin
password=admin
```

### Step 2: Access the Staff Records Portal

The server redirects to `/staff/records` — a staff-only dashboard showing daily permitting totals, the end-of-day checklist, and the "Permit-issuance signing key" which rotates daily.

### Step 3: Read the Flag

The flag is displayed as the signing key at the top of the staff records dashboard:

> **FOR INTERNAL RECONCILIATION ONLY**
>
> **Permit-issuance signing key — 2026-05-12**
>
> `WEBVERSE{2746fa856682b559535755f22ae25818}`

No brute-force, no injection, no bypass — just `admin`/`admin`.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **Default credentials are the most common finding** | Consultants set them for testing; production goes live with the same credentials because someone skipped the binder. |
| **Check `admin`/`admin` on every login form** | It's the single most common default credential pair in web applications. |
| **Footnotes and page metadata are intelligence** | The footer text *"(Default password reset Mar 2017 — see binder.)"* was a direct hint that the default was still in use. |
| **Challenge names are clues** | "Front Counter" — the front desk, the public face, the place where default credentials would be left for the next person to find. |

**Remediation checklist:**
- [ ] Enforce password changes on first login (even for pre-seeded admin accounts)
- [ ] Remove default accounts before going live
- [ ] Audit all consultant-built portals for hardcoded or default credentials
- [ ] Don't leave project binders with credentials on the clerk's shelf

---

*"Wescroft Civic Consulting set the credentials as 'temporary' in 2018. They are still temporary in 2026."*

**Flag:** `WEBVERSE{2746fa85...5818}`
