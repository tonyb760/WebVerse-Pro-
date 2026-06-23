# ☠ INKED — Technique Notes

> **WebVerse Pro · Spoiler-Safe Technique Writeup**
>
> A public-safe writeup for the Inked lab: focused on the web security techniques,
> review mindset, and defensive lessons — without the literal flag, exact solve
> path, target-specific commands, or answer-revealing values.

---

## What This Writeup Does and Does Not

**Inked** is a medium-difficulty WebVerse Pro lab built around a small-business web
stack: public-facing pages, an authenticated staff area, backend API behavior, and
service-to-service trust. This article avoids box-specific answers and instead
explains the underlying techniques at a level useful for future assessments.

| ✅ Included | 🚫 Not Included | 🎯 Goal |
|-------------|----------------|---------|
| Technique concepts, testing heuristics, safe pseudo-examples, security rationale, and remediation guidance | No literal flag, no exact token values, no IP-specific command sequence, no full endpoint list, and no copy-paste solve path | Show how to think about reverse-proxy trust, automated token flows, cross-service auth boundaries, and unsafe template rendering |

> ℹ️ **Lab credit:** Inked is part of [WebVerse Pro](https://webverse.pro).
> If you want the full hands-on experience, play it there first before reading any
> solution-oriented material.

---

## The Technique Chain, Abstracted

The lab rewards careful enumeration and understanding how one layer's assumptions
become another layer's attack surface. At a high level, the chain demonstrates
four reusable web security ideas.

### 🧭 Reverse Proxy Surface Mapping

When several services live behind one IP, the HTTP `Host` header often determines
routing. Different status codes reveal whether a hostname maps to a public site,
login surface, or protected API.

### 📬 Header-Derived URL Trust

Password reset and notification systems sometimes build absolute links from
request headers. If a backend trusts forwarded host/protocol values without
validation, link generation can be redirected.

### 🔑 Cross-Service Token Review

Tokens issued by one frontend may be accepted by another backend if claims like
issuer, audience, or service boundary are missing or weakly enforced.

### 🧪 Template Rendering Risk

User-controlled text must be passed as data, never as template source. If a
backend evaluates stored text through a template engine, expression evaluation
can become code execution.

---

## Testing Mindset Without Spoilers

The following examples are intentionally generic. They show what to test for, not
the exact values from the box.

### 1. Proxy and vhost enumeration

Start by comparing responses across candidate hostnames. Do not only look for
200 responses. A 401 can be more informative than a 404 because it proves a
protected service exists.

```
For each candidate hostname:

  Send request with Host: <candidate-host>

  Record status code, redirect target, content length, title, and server behavior

Prioritize:

  200   → directly reachable surface
  302   → routing clue or canonical host
  401/403 → protected service worth revisiting after auth
  404   → may be default backend or nonexistent route
```

### 2. Header-derived link generation

For password resets and notification workflows, test whether absolute links are
constructed from trusted configuration or from request headers. The safe
application behavior is to ignore client-controlled host values and use a fixed
server-side base URL.

```
Safe pseudo-test:

  Submit a reset request while varying:
    Host: expected-application-host
    X-Forwarded-Host: attacker-controlled-test-host
    X-Forwarded-Proto: http/https

  Then observe, in a controlled lab:
    Does the generated link use the configured application domain?
    Or does it reflect attacker-controlled header values?
```

> ⚠️ **Technique, not answer:** The important lesson is not the exact header
> value. The lesson is that link generation should never depend on untrusted
> request metadata unless a trusted proxy strips and rewrites those headers.

### 3. Token boundary validation

After authentication, inspect token claims and replay boundaries. A token may be
cryptographically valid but semantically invalid for a different service. Good
APIs validate both.

```
Review checklist:

  Decode token header and payload safely.

  Check for:
    alg       → expected and not "none"
    iss       → expected issuer
    aud       → intended service or API
    exp       → expiry enforced
    role/scope → server-side enforced, not UI-only

  Then verify:
    Service A token should not authorize Service B
    unless explicitly intended.
```

### 4. Template injection reasoning

Stored text fields become dangerous if later interpreted by a template engine.
Start with harmless arithmetic or string expressions in a lab. If expressions
evaluate, investigate context and sandboxing before moving further.

```
Generic SSTI workflow:

  Detection phase:
    Submit harmless template expressions as user-controlled text.
    Observe whether output is literal or evaluated.

  Context phase:
    Identify engine family from syntax and behavior.
    Check whether sandboxing is enabled.

  Remediation expectation:
    User text is escaped and rendered as data,
    never compiled as template source.
```

---

## What Defenders Should Fix

| Risk Area | Bad Pattern | Secure Pattern | Detection Idea |
|-----------|-------------|----------------|----------------|
| **Proxy Headers** | Backend trusts `Host` or forwarded host headers for security-sensitive URL generation | Use a configured canonical application URL. Strip untrusted forwarded headers at the edge | Alert on reset or invite flows where forwarded host differs from canonical domain |
| **Token Boundaries** | One service accepts tokens intended for another service | Validate `aud`, `iss`, expiry, scope, and service-specific authorization server-side | Log token audience mismatches and cross-service token replay attempts |
| **Template Rendering** | User-controlled content is compiled as a template | Render user content as escaped data. Enable sandboxing where user templates are truly required | Alert on template metacharacters in stored text fields and template rendering exceptions |
| **Client-Side Discovery** | Frontend code reveals hidden APIs and assumes obscurity protects them | Treat all client-side code as public. Enforce authorization on every backend endpoint | Continuously inventory API routes referenced in JS bundles and compare to access policies |

---

## Why Inked Is a Good Lab

Inked is effective because it makes you chain observations instead of memorizing
payloads. The lesson is the workflow: enumerate routing, understand trust
boundaries, inspect how the frontend talks to the backend, validate token scope,
and treat template engines as interpreters whenever user input crosses that
boundary.

> ✅ **No flag included:** This public version intentionally redacts the flag
> and avoids a direct solve path. If you want to test yourself, play the lab on
> [WebVerse Pro](https://webverse.pro).

---

*☠ INKED — TECHNIQUE NOTES · Public-safe writeup for WebVerse Pro's Inked lab.
No literal flag, no exact answer path.*

[Back to WebVerse Pro](https://webverse.pro)
