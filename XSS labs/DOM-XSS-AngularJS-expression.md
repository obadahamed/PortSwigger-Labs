# 🔀 DOM XSS — AngularJS Expression Injection

![Lab](https://img.shields.io/badge/Lab-DOM%20XSS%20in%20AngularJS-red?style=flat-square)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Solved-brightgreen?style=flat-square)

---

## 📋 Lab Description

This lab contains a DOM-based XSS vulnerability in a AngularJS expression within the search functionality.
AngularJS processes `ng-app` directives and evaluates `{{ }}` expressions — if user input lands inside an AngularJS context, we can execute JavaScript without `<script>` tags.

---

## 🔍 How AngularJS Works

AngularJS scans the DOM for `ng-app` and evaluates expressions like:
```
{{ 7 * 7 }}  →  49
```

If user input is reflected inside an `ng-app` scope, we can inject an expression that executes JS.

---

## 🛠️ Steps to Reproduce

**1. Find the injection point**

Enter a test string in the search box → view page source → confirm the input is reflected inside an `ng-app` element.

**2. Test with a math expression**
```
{{7*7}}
```
If the page shows `49` → AngularJS is evaluating the expression → injection confirmed.

**3. Inject the payload**
```javascript
{{constructor.constructor('alert(1)')()}}
```

This works by accessing the `Function` constructor through the scope chain and executing arbitrary JS.

---

## 💉 Payload Used

```javascript
{{constructor.constructor('alert(1)')()}}
```

| Part | Explanation |
|------|-------------|
| `{{...}}` | AngularJS expression syntax |
| `constructor.constructor` | Accesses the `Function` constructor via the scope chain |
| `('alert(1)')()` | Creates and immediately executes a new function |

---

## ✅ Key Takeaway

> When AngularJS is on the page, you don't need `<script>` tags. The `{{ }}` syntax IS the execution context. Always check for `ng-app` in the source.

---

## 🛡️ Prevention

- Avoid using `$eval`, `$parse`, or unsanitized user input inside AngularJS templates
- Use a strict Content Security Policy (CSP)
- Upgrade to Angular (v2+) which doesn't use client-side templates the same way
