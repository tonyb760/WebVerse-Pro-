# Cookie Cutter — Base64 Is Not Encryption

**Category:** BASICS  
**Difficulty:** EASY  
**Solves:** 84+  
**Flag:** `WEBVERSE{f997a432...e5e2306}`

---

## The Setup

Northbrew Coffee Co., an indie roaster with eleven cafés across the Pacific Northwest, built a loyalty rewards site. The engineering team wanted things to feel "instant" so they baked the user's entire account state into a single browser cookie.

The challenge description practically winks at you: *"They picked an encoding that's easy to read on the wire and easier still to read off it."* That's base64 — not encryption, just a different alphabet.

---

## Recon

The lab site is Northbrew's marketing page — menu, locations, rewards signup. Nothing special on the surface. But one look at the cookies tells the story:

```
nb_session=eyJ0aWVyIjogImdvbGQiLCAiYmVhbl9iYWxhbmNlIjog...
```

The `nb_session` cookie starts with `eyJ` — the dead giveaway of base64-encoded JSON (because `{"` encodes to `eyJ`).

---

## Exploitation

### Step 1: Decode the Cookie

Drop the value into any base64 decoder:

```json
{
  "tier": "gold",
  "bean_balance": 247,
  "next_perk": "free_oat_milk",
  "joined": "2025-08-14",
  "debug": {
    "feature_flag": "rewards_v2_live",
    "internal_ref": "WEBVERSE{f997a432...e5e2306}"
  }
}
```

The flag is sitting right there in `debug.internal_ref`, decoded from the cookie in plain sight.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **Base64 is encoding, not encryption** | Anyone with a browser console can decode it. Don't put secrets in cookies unless they're properly encrypted and signed. |
| **Cookie inspection is step one** | Always check `document.cookie` or the browser DevTools Application tab when investigating a web app. |
| **The `eyJ` pattern** | Base64-encoded JSON almost always starts with `eyJ`. Spotting it saves time. |

**Remediation checklist:**
- [ ] Never store sensitive data (flags, tokens, PII) in client-side cookies
- [ ] If cookies must carry state, sign them with a server-side secret
- [ ] Consider server-side sessions instead of client-stored state

---

*"Northbrew's loyalty program is generous — especially with the flag."*

**Flag:** `WEBVERSE{f997a432...e5e2306}`
