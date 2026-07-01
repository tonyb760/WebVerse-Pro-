# Quikpay Receipts — The Debug Branch That Never Left

**Category:** BASICS  
**Difficulty:** EASY  
**Solves:** 56  
**Flag:** `WEBVERSE{70590f9c...98d5e}`

---

## The Setup

Quikpay is a small payments backend used by indie software shops. The product looks polished — clean receipt pages, a nice customer experience, proper email delivery.

But the engineering lead added a debug branch to the resend handler during a late-night incident and never wrapped it in a feature flag. It shipped to production and silently returns debug data with every receipt resend.

---

## Recon

The lab is a Quikpay merchant dashboard showing recent receipts. Each receipt has a detail view with a "Resend receipt" form that POSTs to `/receipt/<token>/resend`.

Nothing unusual on the page surface — no hidden comments, no secrets in headers, no unusual cookies.

---

## Exploitation

### Step 1: Call the Resend Endpoint

Sending a POST to the resend endpoint with an email:

```http
POST /receipt/qp-r4-9981kl/resend
Content-Type: application/json

{"email": "test@test.com"}
```

### Step 2: Read the Response

The response includes an unexpected `debug` object:

```json
{
  "debug": {
    "exporter_version": "qp-export/2.4",
    "internal_ref": "WEBVERSE{70590f9c...98d5e}",
    "ledger_batch": "lb-2026-w20",
    "reconciled": true
  },
  "ok": true,
  "receipt": { ... }
}
```

The debug handler was left in the production resend endpoint and returns the flag under `debug.internal_ref` every time.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **Debug endpoints are not feature flags** | A debug branch that returns internal state doesn't belong in production, with or without a flag. |
| **Always inspect API response bodies** | Even endpoints that seem harmless (like a "resend" action) can leak data in their responses. |
| **Check every endpoint your app exposes** | Resend, refund, export — any action endpoint could have a debug path. |

**Remediation checklist:**
- [ ] Audit all API routes for debug code paths before shipping
- [ ] Use feature flags properly (or don't ship debug handlers at all)
- [ ] Strip debug objects from production API responses

---

*"Quikpay handles payments. They also handle flags."*

**Flag:** `WEBVERSE{70590f9c...98d5e}`
