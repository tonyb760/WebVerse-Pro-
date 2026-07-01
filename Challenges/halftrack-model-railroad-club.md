# Halftrack Model Railroad Club — The Password That Never Changed

**Category:** AUTH  
**Difficulty:** EASY  
**Solves:** 38+  
**Flag:** `WEBVERSE{c950b3b3...95980}`

---

## The Setup

The Halftrack Model Railroad Club has met in the basement of the Coalridge VFW on the second Tuesday of every month since 1971. Their member portal at `/login` is where members sign in to view dues, roster registration, and the bulletin.

Hollis Kerrigan has been president since 1994. He picked the member-portal admin password himself when the website went up in 1998 and has refused to change it — on principle.

---

## Recon

The login page reveals the username format: `firstinitial + lastname` (lowercase, no spaces). The placeholder "e.g. rcastel" confirms this — that's Rita Castel, the secretary.

The About page lists the officers:

| Name | Office | Username |
|------|--------|----------|
| Hollis Kerrigan | President | `hkerrigan` |
| Marv Delaney | Treasurer | `mdelaney` |
| Rita Castel | Secretary | `rcastel` |
| Jim Worwood | Show Coordinator | `jworwood` |

The challenge description confirms the weak spot: *"There is no rate limit on the login form."* And the password was chosen in 1998 and never changed.

---

## Exploitation

### Step 1: Brute-Force the Password

No rate limit means we can brute-force without worrying about lockouts. The challenge description hints that the admin (`hkerrigan`) chose a weak password in the late 90s.

Using a wordlist of common passwords, `password1` is the winner:

```
POST /login
username=hkerrigan&password=password1
```

### Step 2: Access the Dashboard

Successfully authenticating redirects to `/member/dashboard` — the member portal showing account details, dues status, and the "Layout-Roster Registration Reference" which contains the flag.

### Step 3: Read the Flag

The dashboard displays:

> **LAYOUT-ROSTER REGISTRATION REFERENCE**
>
> `WEBVERSE{c950b3b321ec7275ac6180c96b595980}`

The flag was sitting behind a twenty-six-year-old password that never expired and was never rotated.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **Old passwords are weak passwords** | A password from 1998 (`password1` was already weak then) is almost certainly compromised or guessable. |
| **No rate limit invites brute-force** | Without account lockout or rate limiting, every common password can be tested in seconds. |
| **Username formats leak usernames** | The login page tells you the exact format (`firstinitial + lastname`), and the About page gives you the member names. Together they enumerate every valid account. |
| **Admin accounts get targeted** | The president's account (`hkerrigan`) is the high-value target — it's the one most likely to have a weak, never-changed password. |

**Remediation checklist:**
- [ ] Enforce rate limiting on login forms (e.g., 5 attempts per 15 minutes per IP)
- [ ] Mandate password changes annually, especially for admin/privileged accounts
- [ ] Rotate default or legacy passwords from the original deployment era
- [ ] Don't make username formats public (or obscure them in placeholders)
- [ ] Alert the account owner after successful login from a new IP/device

---

*"Hollis Kerrigan has been president since 1994. He picked the password in 1998. He has refused to change it on principle. The password is `password1`."*

**Flag:** `WEBVERSE{c950b3b3...95980}`
