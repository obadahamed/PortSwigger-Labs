# DOM XSS in `document.write` sink inside a `<select>` element  
**PortSwigger Web Security Academy – Practitioner**

## 🔎 Lab Description
The application uses `document.write()` to dynamically generate a `<select>` element.  
The value of the `storeId` parameter is taken directly from `location.search` and written into the DOM without sanitization.  
Because the input is placed inside a `<select>`/`<option>` context, JavaScript cannot execute unless we break out of the element.

---

## 🧠 Root Cause
The vulnerable code:

```javascript
var store = (new URLSearchParams(window.location.search)).get('storeId');
document.write('<select name="storeId">');

if (store) {
    document.write('<option selected>' + store + '</option>');
}

for (var i = 0; i < stores.length; i++) {
    if (stores[i] === store) continue;
    document.write('<option>' + stores[i] + '</option>');
}

document.write('</select>');
```
Input comes from: ?storeId=...

Written directly into the DOM using document.write()

No encoding or sanitization

Input is placed inside <option> → inside <select> → no JS execution unless we break out

🎯 Exploitation Strategy
Break out of the <select> element using </select>.

Inject a new HTML element capable of executing JavaScript (e.g., <img> with onerror).

Since document.write() writes raw HTML, the payload executes immediately.

💥 Working Payload
Injected into the storeId parameter on the stock checker page:

```javascrept
</select><img src=x onerror=alert(1)>
```
✅ Impact
Arbitrary JavaScript execution in the victim’s browser → DOM-based XSS.

🧩 Key Takeaways
document.write() is inherently dangerous when used with user-controlled input.

Even “safe” HTML contexts like <select> can be escaped.

DOM XSS depends entirely on how the browser processes JavaScript, not the server.
