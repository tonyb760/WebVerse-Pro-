# Vellichor Press — Marginalia the Developers Left Behind

**Category:** BASICS  
**Difficulty:** EASY  
**Solves:** 70+  
**Flag:** `WEBVERSE{5552439a...695945c7}`

---

## The Setup

Vellichor Press is a literary quarterly. Their site has the usual pages — issues, about, contributors, subscribe. The challenge title is *"marginalia"*, which means notes written in the margins of a book.

In web terms: HTML comments.

---

## Recon

The homepage has a clean literary design. Nothing unusual in the visible content, no secrets in cookies or response headers.

But the challenge name tells you exactly where to look: the margins — or in HTML, the comments.

---

## Exploitation

### Step 1: View the Page Source

Viewing the HTML source of the homepage reveals a developer comment in the markup:

```html
<!--
  TODO(elias): remove before final launch. Tracking note while the
  reconciliation script runs against the Stripe export — do NOT push
  this to production after Tuesday's deploy.

  admin shortcut (drafts pane):  /staff/admin?token=stripe-recon-2026
  reconcile internal-ref:        WEBVERSE{5552439a9ec340cf8d6aa8ce695945c7}

  — e.l, 2026-03-12, 11:48 ET. Removed by:                       .
-->
```

A developer named Elias left a TODO comment with an admin token and the flag in an "internal-ref" field. The `Removed by:` line is conspicuously blank — Elias never came back to clean it up.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **HTML comments are the first place to look** | Developers frequently leave notes, TODOs, internal references, and even credentials in HTML comments. They're invisible to users but trivially visible to anyone who views the source. |
| **"TODO: remove before launch" never happens** | If a comment says it should be removed, it almost certainly wasn't. Treat TODO comments in production as security findings. |
| **Challenge names are clues** | "Marginalia" literally means margin notes. The challenge name told you exactly where the flag was hiding. |
| **Check the homepage first** | Comments on the homepage are the easiest to find and the most common hiding spot in BASICS challenges. |

**Remediation checklist:**
- [ ] Strip HTML comments from production builds (use a build step, not developer discipline)
- [ ] Never put secrets, tokens, or internal references in HTML comments
- [ ] Set up CI linting that rejects HTML comment blocks containing keywords like TODO, FIXME, or internal-ref
- [ ] Educate developers that view-source is public — anything in HTML is visible to everyone

---

*"Elias said he'd remove it. He never did."*

**Flag:** `WEBVERSE{5552439a...695945c7}`
