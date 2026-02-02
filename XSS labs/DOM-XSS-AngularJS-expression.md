# DOM XSS in AngularJS Expression Injection  
**PortSwigger Web Security Academy – Practitioner**

## 🔎 Lab Description
The search functionality reflects user input inside an AngularJS expression context.  
Angle brackets `< >` and double quotes `"` are HTML‑encoded, preventing traditional HTML-based XSS.  
However, AngularJS still evaluates expressions inside `{{ ... }}`, allowing JavaScript execution.

---

## 🧠 Root Cause
The page contains:

```html
<body ng-app>
```
This activates AngularJS auto‑expression evaluation.

User input appears inside:

```html
0 search results for 'USER_INPUT'
```
When injecting:

```html
{{7*7}}
```
AngularJS evaluates it → output becomes:

```html
0 search results for '49'
```
This confirms that user input is interpreted as an AngularJS expression.

🎯 Exploitation Strategy
Use AngularJS expression injection instead of HTML tags.

Reach the Function constructor through:

constructor

then constructor.constructor

Build and execute arbitrary JavaScript.

This bypasses:

< encoding

" encoding

HTML tag restrictions

```html
{{constructor.constructor('alert(1)')()}}
```
Explanation:

constructor → gets the constructor of the current object

constructor.constructor → resolves to Function

Function('alert(1)')() → executes JavaScript

✅ Impact
Full JavaScript execution inside the AngularJS context → DOM-based XSS.

🧩 Key Takeaways
AngularJS expression injection is a powerful XSS vector.

No need for HTML tags or angle brackets.

constructor.constructor gives access to the Function constructor → arbitrary JS execution.

Encoding < and " does NOT prevent AngularJS-based XSS.
