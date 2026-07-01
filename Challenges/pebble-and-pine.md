# Pebble & Pine — The Analytics File You Weren't Supposed to Read

**Category:** BASICS  
**Difficulty:** EASY  
**Solves:** 70+  
**Flag:** `WEBVERSE{3e9366d1...5511c7b}`

---

## The Setup

Pebble & Pine is a two-person ceramics studio in the Catskills. Their website sells small-batch mugs and bowls in seasonal runs. The site is clean, minimal, and doesn't track you — or so they claim.

The journal page has a post from 02 FEB 2026 titled *"New website, brother-shaped"* that drops a heavy hint:

> *"Marit's brother Felix — he works in advertising in Brooklyn — built this site over a weekend in January. It's slim, it's quiet, it doesn't track you. He also wrote the analytics file, which apparently we are supposed to ignore for now. He says he'll come back to it. He always says he'll come back to it."*

The analytics file. The one we're *supposed to ignore*. The one Felix *never came back to*.

---

## Recon

The site has standard pages: `/`, `/shop`, `/story`, `/journal`. No hidden HTML comments, no secrets in response headers, no suspicious cookies. The shop is a straightforward product listing for seven ceramic pieces.

The journal page reveals the clue — the analytics file at `/static/js/analytics.js`.

---

## Exploitation

### Step 1: Read the Analytics File

Navigating to the JavaScript file:

```
GET /static/js/analytics.js
```

Returns a compact analytics script with session tracking, scroll-depth monitoring, and outbound link click detection. Standard stuff.

But buried at the bottom, between the scroll tracker and the init function:

```javascript
// ── Internal reconciliation hook — DO NOT REMOVE ──
// The newsletter exporter pulls this constant to tag CSV rows so
// Sasha can match returns to the original campaign batch. Marit's
// brother left it here when he split the analytics file out from
// the campaign tracker. Will remove when the v0.5 split lands.
const INTERNAL_REF = "WEBVERSE{3e9366d14e31a683d7739999c5511c7b}";

// expose for the campaign exporter
window.__pp = window.__pp || {};
window.__pp.ref = INTERNAL_REF;
```

The flag is sitting in a `const INTERNAL_REF` variable, with a comment saying **DO NOT REMOVE** and an excuse about a v0.5 split that was never shipped.

### Step 2: Submit the Flag

The flag is exposed both as the constant value and via `window.__pp.ref` in the browser console.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **Read every JS file** | JavaScript files are often treated as "infrastructure" and skipped during manual review, but they frequently contain hardcoded secrets, debug flags, or internal references. |
| **"Ignore this file" is a red flag** | Any time the narrative tells you not to look somewhere, look there first. It's a planted clue. |
| **The journal/blog told the story** | The flag was hidden in the site's own content — the journal post literally described how the developer left the analytics file in an unfinished state. |
| **Check `/static/js/`** | Always enumerate common JS asset paths. Developers leave comments, debug code, and secrets in static files more often than in server-rendered pages. |

**Remediation checklist:**
- [ ] Audit all static assets (JS, CSS, images) for hardcoded secrets before deploy
- [ ] Strip debug variables and internal reference constants from production builds
- [ ] Don't leave "temporary" hooks in production JS files with comments about future splits
- [ ] Treat `window.__pp`-style global exposure as a code smell

---

*"Pebble & Pine's glazes aren't the only thing you're not supposed to read — their JS files are hiding something too."*

**Flag:** `WEBVERSE{3e9366d1...5511c7b}`
