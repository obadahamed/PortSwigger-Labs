# 🔀 DOM XSS — document.write with select Element

![Lab](https://img.shields.io/badge/Lab-DOM%20XSS%20document.write%20sink-red?style=flat-square)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Solved-brightgreen?style=flat-square)

---

## 📋 Lab Description

This lab contains a DOM-based XSS vulnerability in the search functionality.
The page uses `document.write()` to write a `<select>` element to the DOM using data from `location.search` — the URL query string — without sanitization.

---

## 🔍 How document.write Sink Works

`document.write()` directly writes raw HTML to the page.
If it uses user-controlled data, anything we inject becomes part of the DOM.

```javascript
// Vulnerable code (simplified)
var store = (new URLSearchParams(location.search)).get('storeId');
document.write('<select><option>' + store + '</option></select>');
```

The `storeId` value from the URL goes straight into `document.write()` — no encoding, no sanitization.

---

## 🛠️ Steps to Reproduce

**1. Find the source**

View page source → look for `document.write` using URL parameters like `location.search`.

**2. Confirm the injection point**

Add `?storeId=test123` to the URL → check if `test123` appears in the `<select>` element.

**3. Break out of the `<select>` tag and inject**

```
?storeId="></select><img src=1 onerror=alert(1)>
```

**4. Execution**

The browser closes the `<select>`, renders the `<img>` tag, the `src` fails, and `onerror` fires → `alert(1)` executes.

---

## 💉 Payload Used

```html
"></select><img src=1 onerror=alert(1)>
```

| Part | Explanation |
|------|-------------|
| `">` | Closes the `<option>` value attribute and tag |
| `</select>` | Closes the `<select>` element |
| `<img src=1 onerror=alert(1)>` | Injects a new element that fires JS on error |

---

## ✅ Key Takeaway

> `document.write()` is one of the most dangerous DOM sinks. Anything written via it from a URL parameter is fully attacker-controlled. Always look for it in client-side JS when doing source code review.

---

## 🛡️ Prevention

| Fix | Description |
|-----|-------------|
| Avoid `document.write()` | Use `textContent` or `createElement` instead |
| Sanitize DOM input | Use DOMPurify before writing user data to the DOM |
| CSP | Strict Content Security Policy blocks inline script execution |
