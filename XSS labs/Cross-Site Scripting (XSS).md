
# Cross-Site Scripting (XSS) – Overview & Notes

Cross-Site Scripting (XSS) is one of the most common and impactful vulnerabilities in web applications.  
It occurs when an attacker is able to inject and execute arbitrary JavaScript in a victim’s browser.

This repository contains my personal notes and writeups for XSS labs from PortSwigger Web Security Academy, covering both **Reflected**, **Stored**, and **DOM-based** XSS.

---

## 🔥 What is XSS?

XSS happens when:
- User-controlled input
- Is included in a webpage
- Without proper sanitization or encoding
- Allowing the attacker to execute JavaScript in the victim’s browser

This can lead to:
- Session hijacking  
- Account takeover  
- Keylogging  
- CSRF bypass  
- Full control over the victim’s browser context  

---

## 🧩 Types of XSS

### 1. Reflected XSS
Occurs when malicious input is immediately reflected in the response.

**Example:**
/search?query=<script>alert(1)</script>



### 2. Stored XSS
Payload is stored on the server (e.g., comments, profiles) and executed when viewed by users.

### 3. DOM-Based XSS
The vulnerability exists entirely in **client-side JavaScript**, not the server.

Example sinks:
- `document.write()`
- `innerHTML`
- `location.search`
- `eval()`
- AngularJS expressions (`{{ }}`)

---

## 🛡️ Common XSS Contexts

XSS behavior depends on **where** the input lands:

| Context | Example | Notes |
|--------|---------|-------|
| HTML | `<div>INPUT</div>` | Basic injection |
| Attribute | `<img src="INPUT">` | Requires breaking quotes |
| JavaScript | `var x = 'INPUT';` | Requires breaking string |
| URL | `<a href="INPUT">` | JS URLs possible |
| AngularJS | `{{ INPUT }}` | Expression injection |
| DOM | `document.write(INPUT)` | No server involvement |

---

## 🧠 General Exploitation Strategy

1. **Identify the sink**  
   Where does the input end up? HTML? JS? Attribute? Angular?

2. **Identify the context**  
   Are you inside quotes? Inside a tag? Inside a script?

3. **Break out if needed**  
   Close the tag, close the string, escape the attribute.

4. **Inject JavaScript**  
   Using:
   - `<script>`
   - `<img onerror>`
   - `javascript:` URLs
   - AngularJS expressions
   - Function constructor (`constructor.constructor`)

5. **Trigger execution**  
   Ensure the payload actually runs.

---

## 🧪 Example Payloads

### Basic HTML XSS
```html
<script>alert(1)</script>
```
Attribute Injection
```html
" onerror="alert(1)
```
JavaScript Context
```javascript

';alert(1);//
```
AngularJS Expression Injection
```html
{{constructor.constructor('alert(1)')()}}
```
DOM XSS (document.write)
```html
</select><img src=x onerror=alert(1)>
```

