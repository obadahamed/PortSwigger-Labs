# 🔓 PortSwigger Web Security Academy — Professional Lab Documentation

<p align="center">
  <img src="https://img.shields.io/badge/Platform-PortSwigger%20Academy-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Focus-Web%20Application%20Security-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Labs%20Solved-50%2B-blue?style=for-the-badge"/>
</p>

> **Comprehensive web application security lab documentation** from PortSwigger Web Security Academy.  
> Detailed writeups with vulnerability explanations, exploitation techniques, and defensive mitigations.  
> Each lab includes hands-on proof-of-concept payloads and technical analysis.

---

## 📊 Progress Dashboard

```
VULNERABILITY CATEGORIES
├── 💉 SQL Injection                16 / 18  (89%) ████████░
├── 🔀 Cross-Site Scripting (XSS)   13 / 13  (100%) ██████████
├── 🌐 SSRF                          5 / 7   (71%) ███████░░
├── 📁 Path Traversal               6 / 6   (100%) ██████████
├── ⚙️ Command Injection            5 / 5   (100%) ██████████
├── 📤 File Upload                  7 / 7   (100%) ██████████
├── 🔐 Authentication               8 / 14  (57%) █████░░░░
└── 🛡️ Access Control              10 / 13  (77%) ███████░░
────────────────────────────────────────────────
TOTAL:                        70+ LABS COMPLETED
```

---

## 📚 Vulnerability Categories & Labs

### 💉 SQL Injection (16/18 Labs)

SQL Injection allows attackers to manipulate database queries through unsanitized input.

| Lab | Type | Difficulty | Status |
|-----|------|-----------|--------|
| UNION attack, determining columns | UNION-based | Apprentice | ✅ |
| UNION attack, finding text column | UNION-based | Apprentice | ✅ |
| UNION attack, retrieving data | UNION-based | Apprentice | ✅ |
| UNION attack, multiple values | UNION-based | Apprentice | ✅ |
| Database type & version (Oracle) | DB Enumeration | Apprentice | ✅ |
| Database type & version (MySQL/MSSQL) | DB Enumeration | Apprentice | ✅ |
| Listing database contents (non-Oracle) | DB Enumeration | Apprentice | ✅ |
| Listing database contents (Oracle) | DB Enumeration | Apprentice | ✅ |
| Error-based SQLi | Error-based | Apprentice | ✅ |
| Blind Boolean-based | Blind SQLi | Apprentice | ✅ |
| Blind Boolean with errors | Blind SQLi | Apprentice | ✅ |
| Blind Time-based delays | Blind SQLi | Apprentice | ✅ |
| Blind Time-based with retrieval | Blind SQLi | Apprentice | ✅ |
| Second-order SQLi | Advanced | Practitioner | ✅ |
| Stacked queries | Advanced | Practitioner | ✅ |
| NoSQL injection | Advanced | Practitioner | ✅ |

**Key Techniques:**
- UNION-based extraction
- Error-based exfiltration
- Boolean blind exploitation
- Time-based detection
- Database enumeration

[→ Full SQLi Documentation](./SQLi_LABS.md)

---

### 🔀 Cross-Site Scripting (XSS) (13/13 Labs) ✅

XSS allows injection of malicious JavaScript that executes in victims' browsers.

| Lab | Type | Difficulty | Status |
|-----|------|-----------|--------|
| Reflected XSS, HTML context | Reflected | Apprentice | ✅ |
| Stored XSS, HTML context | Stored | Apprentice | ✅ |
| DOM XSS (document.write) | DOM-based | Apprentice | ✅ |
| DOM XSS (innerHTML) | DOM-based | Apprentice | ✅ |
| DOM XSS (jQuery href) | DOM-based | Apprentice | ✅ |
| DOM XSS (jQuery selector) | DOM-based | Apprentice | ✅ |
| Reflected XSS in attribute | Reflected | Apprentice | ✅ |
| Stored XSS in href attribute | Stored | Apprentice | ✅ |
| Reflected XSS in JavaScript string | Reflected | Apprentice | ✅ |
| DOM XSS in select element | DOM-based | Practitioner | ✅ |
| AngularJS expression injection | DOM-based | Practitioner | ✅ |
| Reflected DOM XSS | Reflected DOM | Practitioner | ✅ |
| Cookie theft via XSS | Exploitation | Practitioner | ✅ |

**Exploitation Methods:**
- Reflected injection
- Stored persistence
- DOM manipulation
- Event-based triggers
- Cookie exfiltration
- Session hijacking

[→ Full XSS Documentation](./XSS_LABS.md)

---

### 🌐 Server-Side Request Forgery (SSRF) (5/7 Labs)

SSRF exploits trust relationships to access internal systems from server-side requests.

| Lab | Difficulty | Status |
|-----|-----------|--------|
| Basic SSRF against localhost | Apprentice | ✅ |
| SSRF with blacklist filter bypass | Apprentice | ✅ |
| SSRF with whitelist bypass | Apprentice | ✅ |
| SSRF via OpenRedirect | Practitioner | ✅ |
| Blind SSRF with OOB | Practitioner | ✅ |
| SSRF on Elastic Beanstalk metadata | Advanced | ❌ |
| Blind SSRF with time delays | Advanced | ❌ |

**Attack Vectors:**
- Local resource enumeration
- Internal service access
- Metadata service exploitation
- Out-of-band techniques
- Blind detection

[→ Full SSRF Documentation](./SSRF_LABS.md)

---

### 📁 Path Traversal (6/6 Labs) ✅

Path traversal allows directory escape to access unauthorized files.

| Lab | Difficulty | Status |
|-----|-----------|--------|
| File path traversal, simple case | Apprentice | ✅ |
| Traversal with absolute path bypass | Apprentice | ✅ |
| Traversal with nested encoding | Apprentice | ✅ |
| Traversal with validation stripping | Practitioner | ✅ |
| Traversal with null byte | Practitioner | ✅ |
| Traversal with double encoding | Practitioner | ✅ |

**Bypass Techniques:**
- Relative path sequences (`../`)
- Absolute paths (`/etc/passwd`)
- URL encoding (`%2e%2e/`)
- Double encoding (`%252e%252e/`)
- Null byte injection (`..%00`)

[→ Full Path Traversal Documentation](./File-Path-Traversal_LABS.md)

---

### ⚙️ OS Command Injection (5/5 Labs) ✅

Command injection allows arbitrary OS command execution through user input.

| Lab | Difficulty | Status |
|-----|-----------|--------|
| OS command injection, simple case | Apprentice | ✅ |
| Blind OS command injection | Apprentice | ✅ |
| Blind with output redirection | Apprentice | ✅ |
| Blind with OOB | Practitioner | ✅ |
| Time-based blind detection | Practitioner | ✅ |

**Exploitation Methods:**
- Command separators (`;`, `|`, `||`, `&&`, `` ` ``, `$()`)
- Time-based detection
- Output redirection
- Out-of-band channels

[→ Full Command Injection Documentation](./OS-Command-Injection.md)

---

### 📤 File Upload (7/7 Labs) ✅

File upload vulnerabilities allow arbitrary file storage/execution.

| Lab | Difficulty | Status |
|-----|-----------|--------|
| Remote code execution via unrestricted upload | Apprentice | ✅ |
| Polyglot file (PHP/JPEG) | Apprentice | ✅ |
| Extension blacklist bypass | Apprentice | ✅ |
| Null byte in filename | Apprentice | ✅ |
| Upload path traversal | Practitioner | ✅ |
| Race condition in file processing | Practitioner | ✅ |
| SVG with embedded JavaScript | Practitioner | ✅ |

**Bypass Techniques:**
- MIME type spoofing
- Polyglot files
- Null byte injection
- Double extensions
- Path traversal
- Race conditions

[→ Full File Upload Documentation](./File-Upload-Vulnerabilities_LABS.md)

---

### 🔐 Authentication (8/14 Labs)

Authentication flaws allow unauthorized access or account takeover.

| Lab | Difficulty | Status |
|-----|-----------|--------|
| Username enumeration | Apprentice | ✅ |
| Brute-force attack | Apprentice | ✅ |
| Password reset flaws | Apprentice | ✅ |
| Insecure email verification | Practitioner | ✅ |
| OAuth token misuse | Practitioner | ✅ |
| Multi-factor bypass | Practitioner | ✅ |
| Session fixation | Practitioner | ✅ |
| JWT signing bypass | Practitioner | ✅ |

**Techniques:**
- Credential brute-force
- Account enumeration
- Token manipulation
- Session hijacking
- MFA bypass

---

### 🛡️ Access Control (10/13 Labs)

Access control bypasses allow unauthorized resource access.

| Lab | Difficulty | Status |
|-----|-----------|--------|
| Unprotected admin functionality | Apprentice | ✅ |
| Parameter-based access control | Apprentice | ✅ |
| Horizontal privilege escalation | Apprentice | ✅ |
| Vertical privilege escalation | Practitioner | ✅ |
| Insecure direct object reference (IDOR) | Practitioner | ✅ |
| Vertical via platform misconfiguration | Practitioner | ✅ |
| Method-based bypass | Practitioner | ✅ |
| Multi-step bypass | Advanced | ✅ |
| Referer-based access control | Advanced | ✅ |
| JWT in cookie | Advanced | ✅ |

---

## ���️ Tools & Techniques

### Essential Tools

| Tool | Purpose | Usage |
|------|---------|-------|
| **Burp Suite** | Web proxy & security testing | Request interception, payload crafting |
| **Firefox DevTools** | Browser inspection | DOM analysis, network inspection |
| **Kali Linux** | Penetration testing OS | Exploit scripts, command-line tools |
| **curl** | HTTP client | Command-line request testing |
| **Python** | Script automation | Exploit development, automation |

### Attack Methodology

```
1. RECONNAISSANCE
   └─ Map application functions
   ├─ Identify input points
   └─ Check response behavior

2. VULNERABILITY SCANNING
   ├─ Test each input point
   ├─ Analyze error messages
   └─ Look for patterns

3. EXPLOITATION
   ├─ Craft payload
   ├─ Test evasion techniques
   └─ Verify execution

4. DOCUMENTATION
   ├─ Record payload
   ├─ Document technique
   └─ Note defensive measures
```

---

## 📋 Learning Path

### Week 1: Foundations
- ✅ SQL Injection basics
- ✅ XSS fundamentals
- ✅ Path traversal

### Week 2: Advanced Techniques
- ✅ Blind SQLi (Time-based, Boolean)
- ✅ DOM XSS vectors
- ✅ Command injection

### Week 3: Complex Scenarios
- ✅ SSRF exploitation
- ✅ File upload bypasses
- ✅ Authentication flaws

### Week 4+: Real-World Applications
- 🔄 Authentication mechanisms
- 🔄 Access control models
- 🔄 Business logic exploitation

---

## 💡 Key Learnings

### SQL Injection
- **Reconnaissance:** Determine database type, version, structure
- **Extraction:** Use UNION-based, error-based, or blind techniques
- **Escalation:** Access file system, OS command execution

### XSS
- **Identification:** Find injection point and context (HTML, JS, attribute)
- **Bypass:** Encode detection, use event handlers
- **Impact:** Cookie theft, session hijacking, malware distribution

### SSRF
- **Discovery:** Find server-side request functionality
- **Bypass:** Whitelist/blacklist evasion, protocol manipulation
- **Impact:** Internal service access, metadata exposure

---

## 🔐 Defensive Measures

### SQL Injection Prevention
```
✅ Parameterized queries / Prepared statements
✅ Input validation & whitelist
✅ Principle of least privilege
✅ WAF rules for SQLi patterns
```

### XSS Prevention
```
✅ Output encoding (context-aware)
✅ Content Security Policy (CSP)
✅ HttpOnly & Secure flags on cookies
✅ Input validation
```

### General Web Security
```
✅ HTTPS/TLS encryption
✅ Strong authentication (MFA)
✅ Proper error handling
✅ Security headers
✅ Regular patching
```

---

## 📖 Documentation Structure

Each vulnerability category has dedicated documentation:

- **[SQLi Labs](./SQLi_LABS.md)** — 16 techniques with cheat sheets
- **[XSS Labs](./XSS_LABS.md)** — 13 types with decision trees
- **[SSRF Labs](./SSRF_LABS.md)** — 5 attack vectors
- **[Command Injection](./OS-Command-Injection.md)** — 5 exploitation methods
- **[File Upload](./File-Upload-Vulnerabilities_LABS.md)** — 7 bypass techniques
- **[Path Traversal](./File-Path-Traversal_LABS.md)** — 6 evasion methods

---

## 🎯 My Approach to Lab Solving

For every lab, I follow this systematic process:

```
1. Read the lab description carefully
   └─ Understand the application context

2. Explore the application
   ├─ Identify all input points
   ├─ Analyze responses
   └─ Look for hints

3. Formulate hypothesis
   ├─ What vulnerability might exist?
   └─ Where could it be?

4. Test the hypothesis
   ├─ Start with simple payloads
   ├─ Escalate gradually
   └─ Observe all responses

5. Exploit the vulnerability
   ├─ Craft working payload
   ├─ Verify execution
   └─ Document everything

6. Learn from the lab
   ├─ Why was it vulnerable?
   ├─ How would defenders prevent it?
   └─ What patterns should I watch for?
```

---

## ⚠️ Educational Use Only

> **IMPORTANT:** All content in this repository is for **educational purposes only**. These techniques should only be used on:
> - Authorized penetration testing engagements
> - PortSwigger Web Security Academy labs
> - Systems you own or have explicit permission to test
>
> **Unauthorized access** to computer systems is illegal. Always obtain written authorization before any security testing.

---

## 🔗 External Resources

### Documentation
- [OWASP Top 10](https://owasp.org/www-project-top-ten/) — Web application security risks
- [OWASP Cheat Sheets](https://cheatsheetseries.owasp.org/) — Security best practices
- [PortSwigger Documentation](https://portswigger.net/web-security) — Official guides

### Practice Platforms
- [PortSwigger Web Security Academy](https://portswigger.net/web-security) — Official labs
- [HackTheBox](https://www.hackthebox.com/) — Real-world scenarios
- [TryHackMe](https://www.tryhackme.com/) — Guided labs

### Tools
- [Burp Suite Community](https://portswigger.net/burp/communitydownload) — Free version
- [OWASP ZAP](https://www.zaproxy.org/) — Open-source scanner
- [SQLMap](http://sqlmap.org/) — SQL injection automation

---

## 🤝 Connect

[![GitHub](https://img.shields.io/badge/GitHub-obadahamed-181717?style=for-the-badge&logo=github)](https://github.com/obadahamed)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Obada%20Hamed-0077B5?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/obadahamed)
[![TryHackMe](https://img.shields.io/badge/TryHackMe-NEPHOS-red?style=for-the-badge&logo=tryhackme)](https://tryhackme.com/p/NEPHOS)

---

**Last Updated:** June 2026  
**Author:** Obada Hamed  
**Status:** 🟢 Actively Updated

*This repository represents my journey through hands-on web application security training, documenting each vulnerability, technique, and lesson learned along the way.*
